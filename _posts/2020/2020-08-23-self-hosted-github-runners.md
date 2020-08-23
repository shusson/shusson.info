# Self-hosted ubuntu github runners

__23/08/2020__

![potato](/assets/potato.png)

Currently, github runners are hosted on azure [Standard_DS2_v2 VMs](https://docs.github.com/en/actions/reference/virtual-environments-for-github-hosted-runners). They have 2vCPU 7GB RAM and 14GB SSD.

You can self-host a runner by installing the runner service on your own infrastructure. Self-hosted runners incur no costs on github and allow you to leverage the actions api. Of course you'll be paying for your own infrastructure, but often this will be cheaper than burning through github minutes. If you've got a couple idle ubuntu boxes lying around the office it's an easy win.

I would recommend using a ramdisk for the workers `_work` dir as it will dramatically increase IO. Since our tests are ephemeral anyway, we don't care if there is data loss in the `_work` directory. Any important files can be persisted using [github artifacts](https://docs.github.com/en/actions/configuring-and-managing-workflows/persisting-workflow-data-using-artifacts).

If your workflow is a [container-job](https://docs.github.com/en/actions/configuring-and-managing-workflows/about-service-containers#running-jobs-in-a-container), then the only dependencies are ubuntu and docker. Assuming you've got stock ubuntu 18.04 installed:

```bash
sudo apt update
sudo apt install -y docker.io
sudo systemctl start docker
sudo systemctl enable docker
# requires reboot
sudo usermod -a -G docker fred


# create a ramdisk for the runners work dir
mkdir /home/fred/ramdisk
sudo chown -R fred /home/fred/ramdisk

# requires reboot
echo "tmpfs /home/fred/ramdisk tmpfs   defaults,size=3000M,nosuid,uid=1000,gid=1000   0 0" | sudo tee -a /etc/fstab

# set up the github runner
# in this example we are installing a runner to a repository, you can also install it for a org
# more info https://docs.github.com/en/actions/hosting-your-own-runners/adding-self-hosted-runners
mkdir actions-runner
cd actions-runner
# note the version will most likely change in the future
curl -O -L https://github.com/actions/runner/releases/download/v2.272.0/actions-runner-linux-x64-2.272.0.tar.gz
tar xzf ./actions-runner-linux-x64-2.272.0.tar.gz
# you can find the token here: https://github.com/<org>/<repo>/settings/actions/add-new-runner
./config.sh --url https://github.com/<org>/<repo> --token "${GITHUB_TOKEN}" --name "${RUNNER_NAME}" --work /home/fred/ramdisk  --unattended  --replace
sudo ./svc.sh install
sudo ./svc.sh start
sudo ./svc.sh status
sudo reboot
```

And that's it. You should now see the runner in your repo: e.g `https://github.com/<org>/<repo>/settings/actions`

To tie your workflow to a self-hosted runner, use a label e.g

```
jobs:
    server_tests:
        runs-on: [self-hosted, enabled]
```

You might be wondering if you can host the github runner in a docker container. Unfortunately not all features are supported yet, so depending on your workflow you might have [issues](https://github.com/actions/runner/issues/406).
