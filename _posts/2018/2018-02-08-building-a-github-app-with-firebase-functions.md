# Building a Github App with Firebase functions

__08/02/2018__

![bots](https://imgs.xkcd.com/comics/twitter_bot.png)

At work I wanted to automate some of the things we commonly do on github. For example add a comment with the link to the Jira tickets related to the branch. Also during reviews there were some patterns that would commonly crop up like committing `it.only` to a Jest test, which, in our team, we would never expect to be committed. I am not one for re-inventing the wheel, but in this case it did seem like a simple github bot would improve our efficiency on github. And I couldn't find anything that would meet our needs.

Github Apps provide a nice way to automate various activities on Github without using personal authentication. Firebase functions let you create services without having to worry about infrastructure.

## Firebase set-up

First sign up to [firebase](https://firebase.google.com/). You'll need the pay as you go plan (Blaze) because we'll be making external requests. Unless your bot is recieving 1000s of requests, you won't have to pay anything because of the free tier.

Install firebase tools:

```bash
npm install -g firebase-tools
```

Set up your app:

```bash
mkdir mybot
cd mybot
firebase init
```

Follow the prompts and make sure you enable functions. For this post I'm using Typescript.

Enable the helloworld function and deploy

```bash
vi functions/src/index.ts
firebase deploy
```

Firebase should output a message like this:

```bash
✔  functions[github]: Successful update operation.
Function URL (github): https://us-central1-<projectname>.cloudfunctions.net/<functionname>
```

Done! You have a running service that you can query. If you browse to your function URL above you should see `Hello from Firebase!`.

## Integrating the Github App

Follow this [guide](https://developer.github.com/apps/building-github-apps/creating-a-github-app/) to create a Github App.

Add your firebase function URL for the webhook setting.

For now set the Homepage URL and User authorization callback URL to your github repo.

Once the App is created, grab the following things from the Github App settings:

- app id
- private key (pem) (Will need to be converted into one giant string with '\\n' separating the new lines)
- The webhook secret from github

Add these to the firebase config like so:

```bash
firebase functions:config:set github.appid=xxx
```

Validate the webhook:

```typescript
const cipher = 'sha1';
const signature = request.headers['x-hub-signature'];

const hmac = crypto.createHmac(cipher, functions.config().github.secret)
    .update(JSON.stringify(request.body, null, 0))
    .digest('hex');

const expectedSignature = `${cipher}=${hmac}`;

if (!secureCompare(signature, expectedSignature)) {
    throw new Error('x-hub-signature did not match');
}
```

Generate a JWT:

```typescript
const appId = functions.config().github.appid;
const pem = functions.config().github.pem;
// firebase config will strip out `\`, so we store the pem with extra `\`
// and strip it out here
const jwt = jsonwebtoken.sign({iss: appId},
    pem.replace(/\\n/g, '\n'), {
        algorithm: 'RS256',
        expiresIn: '10m'
    });
```

Request an access token:

```typescript
const token = rp({
    url: `https://api.github.com/installations/${request.body.installation.id}/access_tokens`,
    json: true,
    headers: {
        'Authorization': 'Bearer ' + token,
        'User-Agent': 'FredBot',
        'Accept': 'application/vnd.github.machine-man-preview+json'
    },
    method: 'POST'
});
```

## Start automating!

To get the comments in a PR:

``` typescript
export const github = functions.https.onRequest(async (req: Request, res: Response) => {
    rp({
        json: true,
        headers: defaultHeaders(token),
        method: 'GET',
        url: req.body.pull_request.comments_url,
    })
}

```

To get the diff of a PR:

```typescript
export const github = functions.https.onRequest(async (req: Request, res: Response) => {
    rp({
        headers: {
            Authorization: "token " + token,
            "User-Agent": userAgent,
            Accept: "application/vnd.github.v3.diff"
        },
        method: "GET",
        url: req.body.url
    });
}
```

We run simple regex checks on the diff to detect common mistakes that our tooling doesn't pick up, like the Jest `it.only` pattern.
