# Using Firebase and Auth0 together

__24/10/2017__

![archer](/assets/archer.jpg)

Firebase has a decent [drop in authentication](https://firebase.google.com/docs/auth/)
solution, but it lacks many features that a dedicated identity service like
[Auth0](https://auth0.com/) can provide. So lets join the two. A live demo is
available [here](https://custom-auth-d9c94.firebaseapp.com/) and the source is
available on [github](https://github.com/shusson/firebase-custom-auth).

## Prerequisites

- Create a web app (the demo uses Angular).
- Set up Auth0 authentication for your web app. If you're using Angular, Auth0 has a good guide on [getting started](https://auth0.com/docs/quickstart/spa/angular2/01-login).
- [Set up Firebase](https://firebase.google.com/docs/web/setup).

## Create the token on Auth0

Firebase requires a JWT token for custom authentication, we'll use an Auth0
[rule](https://auth0.com/docs/rules/current) to create the token and sign it.

Auth0 Rule:

```javascript
function (user, context, callback) {
    const privateKey = 'FIREBASE_PRIVATE_KEY';
    const email = 'FIREBASE_SERVICE_EMAIL';
    const now = Math.floor(new Date().getTime() / 1000);
    const data = {
        iss: email,
        sub: email,
        aud: "https://identitytoolkit.googleapis.com/google.identity.identitytoolkit.v1.IdentityToolkit",
        iat: now,
        exp: now + 3600,
        uid: user.user_id
    };
    const firebaseToken = jwt.sign(data, privateKey, {expiresInMinutes: 60, algorithm: 'RS256'});
    context.idToken['http://foo/firebaseToken'] = firebaseToken;
    callback(null, user, context);
}
```

You can generate the key and email by following this [guide](https://firebase.google.com/docs/admin/setup?authuser=0)

## Use the token on the client

You should now see the Firebase token in the response to a user authentication.
We'll use the token to sign into firebase:

```javascript
const ft = authResult.idTokenPayload["http://foo/firebaseToken"];
firebase.auth().signInWithCustomToken(ft).then((user: firebase.User) => {
    ...
    // do stuff
}).catch(function(e) {
    console.log(e);
});
```

In the demo we persist the token in local storage so that the user doesn't have
to log in while the token is still valid.

Now that you're authenticated you can use other Firebase services like Firestore.
In the demo we use Firestore to [retrieve or create user details](https://github.com/shusson/firebase-custom-auth/blob/master/src/app/app.component.ts#L22)
based on whether we've seen the user before. We restrict the user to only
their own details by setting up a Firestore rule:

 ```text
service cloud.firestore {
  match /databases/{database}/documents {
    match /users/{userId} {
      allow read, write: if request.auth.uid == userId;
    }
  }
}
 ```
