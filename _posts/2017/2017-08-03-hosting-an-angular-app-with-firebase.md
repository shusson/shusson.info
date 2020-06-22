# Hosting an Angular app with Firebase

__03/08/2017__

![trace](/assets/firebase.jpg)

Firebase hosting comes with a generous
[free tier](https://firebase.google.com/pricing/) and great tooling.
Keep in mind that if you expect any serious traffic, you will have to
upgrade to a paid plan and at that point it's probably worth
considering dedicated hosting unless you are using any of the other
Firebase services.

## Prerequisites

 - [angular-cli](https://github.com/angular/angular-cli)

## Process

Create a new application with the CLI:

``` bash
ng new myapp
cd myapp
```

Build the app:

``` bash
ng build --target=production --aot
```

The CLI stores build articfacts in a `dist` directory by default. We'll
use that to deploy our app. We use the
[target](https://github.com/angular/angular-cli/wiki/build#bundling--tree-shaking)
and [aot](https://angular.io/guide/aot-compiler)
options so that the CLI does some optimizations.

Install the firebase tools:

``` bash
npm install -g firebase-tools
```

Login to firebase:

``` bash
firebase login
```

Initialize firebase for the project. You'll be prompted to set the
`public` directory, make sure it is set to the angular-cli
build directory, which is `dist` by default:

``` bash
firebase init
```

Deploy the app:

``` bash
firebase deploy
```

And that's it! You should see something like:

``` bash
=== Deploying to 'xxx'...

i  deploying hosting
i  hosting: preparing dist directory for upload...
✔  hosting: x files uploaded successfully
i  starting release process (may take several minutes)...

✔  Deploy complete!

Project Console: https://console.firebase.google.com/project/xxx/overview
Hosting URL: https://xxx.firebaseapp.com
```
