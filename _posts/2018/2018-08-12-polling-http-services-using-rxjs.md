# Polling http services using rxjs

__12/08/2018__

![polling](https://imgs.xkcd.com/comics/data.png)

Sometimes you just gotta poll. With observables, polling is quite declarative and concise. In this example we are going to start a job on a server and poll until the job is completed. During the waiting we can do side effects like update a progress bar.

Example:

```typescript

const options = {};
const url = "SOME_URL";
const pollingUrl = "SOME_POLLING_URL";

// this is the http request that will start the job on the server.
const startJob = this.http.post(
    url,
    options
);

/**
 * declare a timer observable that fires every 500ms and starts at 0ms.
 * on every event poll the server and subscribe to the result using switchMap.
 * switchMap will also cancel pending requests if
 * nothing is returned within the timer period.
 * the final step is to filter out events that are not completed.
 */
const polling = (job: JobDTO) => {
    return timer(0, 500).pipe(
        switchMap(() => this.http.get(pollingUrl)),
        // we could add a tap here to update any progress indicators.
        filter((job: any) => job.state === "COMPLETED"),
        take(1)
    );
};

/**
 * start the job, and begin polling.
 * to map the starting observable to the polling observable we use concatMap.
 * and there's a timeout so that we don't wait indefinitely.
 * we transform the observable to a promise so that we can `await` it.
 */
return await startJob
    .pipe(
        concatMap((job) => polling(job).pipe(timeout(5000)))
        tap(
            (job) => {
                // some side effect like:
                // window.location.href = `${environment.SERVER_API_URL}${job.result.link}`
            }
        )
    ).toPromise();
```
