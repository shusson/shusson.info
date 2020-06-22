# Non-durable postgres for e2e tests

__08/01/2020__

![honest](https://imgs.xkcd.com/comics/honest.png)

During e2e testing we can sacrifice postgres [durability in favour of performance](https://www.postgresql.org/docs/12/non-durability.html).
Durability guarantees the saving of data even if the server crashes or loses power,
which is not usually needed during e2e tests. It's arguable that e2e tests should test a system
as close to production as possible, but I think if you need to speed up your tests this is a worthy trade-off.

Simply add the following to the `/var/lib/postgresql/data/postgresql.conf` file:

```text
fsync = off
synchronous_commit = off
full_page_writes = off
```

Note that `fsync` can only be set in the `postgresql.conf` or on the server command line.

Using these settings we were able to decrease our test running time by ~20%.
Given that our e2e tests are not all db focused, I think this is a good result.

A non durable postgres docker image can be found here: https://hub.docker.com/repository/docker/shusson/nd-postgres
