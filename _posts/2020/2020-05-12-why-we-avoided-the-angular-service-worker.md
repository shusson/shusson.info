# Why we avoided the angular service worker

__12/05/2020__

![refresh](https://imgs.xkcd.com/comics/refresh_types.png)

We were looking at using the [Angular service worker](https://angular.io/guide/service-worker-intro) to provide
app integrity during client updates. Since Angular is a single page application (SPA), clients may have loaded the old version of the application in a browser tab and will not get
the new version until they refresh or open a new tab. Cache busting with a webserver like nginx doesn't help, since navigation inside the application is intercepted by Angular.

When we deploy a new version of our angular application, we want to ensure our users:

- Do not get any errors if they continue to use the application (even if it is the old version).
- Are upgraded to the latest angular application or are prompted to upgrade.

Enter the Angular service worker, which [promises](https://angular.io/guide/service-worker-intro#service-workers-in-angular):

```
to optimize the end user experience of using an application over a slow or unreliable network connection, while also minimizing the risks of serving outdated content.

The Angular service worker's behavior follows that design goal:

- Caching an application is like installing a native application. The application is cached as one unit, and all files update together.
- A running application continues to run with the same version of all files. It does not suddenly start receiving cached files from a newer version, which are likely incompatible.
- When users refresh the application, they see the latest fully cached version. New tabs load the latest cached code.
- Updates happen in the background, relatively quickly after changes are published. The previous version of the application is served until an update is installed and ready.
- The service worker conserves bandwidth when possible. Resources are only downloaded if they've changed.
```

We aren't interested in offline capabilities, but maintaining application integrity sounds great. Note that I'm evaluating the Angular service worker for application integrity, not service workers in general.

The docs have a good [description](https://angular.io/guide/service-worker-devops#service-worker-and-caching-of-app-resources) of how the Angular service worker achieves these goals:

```
Conceptually, you can imagine the Angular service worker as a forward cache or a CDN edge that is installed in the end user's web browser
```

## Pros

- Installing the angular service worker into our app was [simple and straight forward](https://angular.io/guide/service-worker-getting-started).
- The service worker will indeed maintain the app version as stated in it's design goals.
- Adding the logic to prompt the user to upgrade was also [straight forward](https://angular.io/guide/service-worker-communications#checking-for-updates) (although the `isStable` check seems odd and bound to cause sneaky issues).

## Deal breakers:

- Service workers don't work behind a [redirect](https://angular.io/guide/service-worker-devops#changing-your-apps-location)

The solution as suggested by the docs in such a situation:

```
instead, you must serve the contents of safety-worker.js at the URL of the Service Worker script you are trying to unregister, and must continue to do so until you are certain all users have successfully unregistered the old worker. For most sites, this means that you should serve the safety worker at the old Service Worker URL forever.
```

This is a pretty big pot-hole to fall into and one I would rather not have to consider when deploying our app. Especially since it is likely we will change
the location of our app at some point. I feel like this problem as a whole exposes the main weakness with the angular service worker, which is if it's broken, the only way to fix it is to update the service worker, but the service worker is responsible to update itself. I also feel like there some nasty security implications here as well, e.g a compromised angular service worker has complete control over the contents of your application on the users browser until the user hard refreshes or removes the service worker manually.

- Can only use the service worker with [production settings](https://angular.io/guide/service-worker-getting-started#serving-with-http-server)

From a development perspective this makes using the angular service worker a real pain and more likely to cause some horrible bug that slips through into production. Which as pointed out above, is likely to cause something which can only be fixed by updating the angular service worker or by using one of the [fail-safes](https://angular.io/guide/service-worker-devops#fail-safe). Alternatively tell all your users to hard refresh or manually unregister the service worker.

## Conclusion

Despite the potential, the Angular service worker is just not worth the extra complexity it adds to our application. We will be fixing our application integrity problems in other not so great ways probably in a different blog post.


