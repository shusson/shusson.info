# Error handling with Angular

__16/10/2018__

An important step to creating a solid Angular application is setting up error handling.
There's two types we need to handle, HTTP errors and javascript errors. There are a set of HTTP errors that can usually be handled generically like 500s or 403s. 400s should be passed on to the component that made the HTTP request so that it can handle the error appropriately. Javascript errors are not expected but can (will) happen so we need to have a friendly way of handling them.

## Handling javascript errors

Angular comes with an error handler that can be overridden. When this function is called it is safe to assume the application is in a broken state. This means we can't safely use any angular libraries and to show the error page, the safest most reliable thing to do is to use the browser api directly to navigate to an error page.

Another common handling behavior is to send the error to an external HTTP service like Sentry. If this is done, make sure you add a timer before you navigate away to the error page. Don't wait on the response of the error notification because that in itself might error.

```typescript
@Injectable({
    providedIn: "root"
})
export class CustomErrorHandler implements ErrorHandler {
    constructor(private injector: Injector) {}

    handleError(error: Error): void {
        // safest thing is to redirect to an error page.
        // we can use local storage to cache the last error
        window.location.href = `${window.location.origin}/error`
    }
}
```

## Intercepting HTTP errors

Angular also provides a HTTP interceptor. It's pretty straightforward to setup. Notice that we propagate errors that are we don't handle and that we setup some `IGNORED_URLS` that skip the generic handling. This is important because some pages, like the login page, handle certain HTTP errors like `UNAUTHORIZED` in a special way.

```typescript
export class HttpErrorInterceptor implements HttpInterceptor {
    constructor(
        private loggerService: LoggerService,
        private router: Router
    ) {}

    intercept(request: HttpRequest<any>, next: HttpHandler): Observable<any> {
        return next.handle(request).catch((error: any) => {
            if (!(error instanceof HttpErrorResponse)) {
                return throwError(error);
            }

            // allows certain urls to avoid the interceptor
            // sometimes you don't want to handle errors generically.
            if (IGNORED_URLS.includes(request.url)) {
                return throwError(error);
            }

            switch (error.status) {
                case HttpStatus.UNAUTHORIZED:
                    this.router.navigate(["login"], queryParams);
                    return of();

                case HttpStatus.INTERNAL_SERVER_ERROR:
                    this.showErrorPage(error, errorReport, ["error"]);
                    return of();

                default:
                    this.loggerService.warn(`Propagating HttpError ${error.status}`);
                    return throwError(error);
            }
        });
    }
}
```
