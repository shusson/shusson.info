# Generic logging on a class using decorators in Typescript

__01/06/2018__

![decorators](https://gist.githubusercontent.com/shusson/622e62166879d1a9b85ba0b0d01345a7/raw/0549e5598f386ede6e42ad60a44f9004810b308d/decorators-decorators-everywhere.jpg)

When debugging an application it's nice to have some extra logging when a full stacktrace is not available. For example slow performance of a certain endpoint or an unhandled promise exception, where the stacktrace is garbage. Ideally we wanted something that was generic, ie we didn't want to have to manually add log lines to methods.

The output should look something like:

```bash
2018-12-29T12:27:51.832Z debug: [AuthService][login]	Entering
2018-12-29T12:27:51.844Z debug: [AuthService][login]	Exiting 12ms
```

To achieve this we ended up using [Typescript decorators](https://www.typescriptlang.org/docs/handbook/decorators.html). Keep in mind that decorators are still an experimental feature, but since Angular 2+ makes heavy use of them, I expect they will become standard at some point.

## Implementation

Pseudocode:

```text
for each property of the class
    if property is a function
        wrap function with added logs before and after the function execution
        re-assign class property to the newly wrapped function
```

In code the decorator looks like:

```typescript
export function Loggable() {
    return (target: Function) => {
        for (const propertyName of Object.getOwnPropertyNames(target.prototype)) {

            const descriptor = Object.getOwnPropertyDescriptor(target.prototype, propertyName);

            if (!descriptor) {
                continue;
            }

            const originalMethod = descriptor.value;

            const isMethod = originalMethod instanceof Function;

            if (!isMethod) {
                continue;
            }

            descriptor.value = function(...args: any[]) {
                console.log(`[${target.name}][${propertyName}] Entering`);

                const now = Date.now();
                const result = originalMethod.apply(this, args);

                const exitLog = () => {
                    console.log(`[${target.name}][${propertyName}] Exiting ${Date.now() - now}ms`);
                };

                // work around to support async functions.
                if (typeof result === "object" && typeof result.then === "function") {
                    const promise = result.then(exitLog);

                    // we defer responsibility to the caller of the method to handle the error.
                    // but we need to catch the error otherwise we will get an unhandled error.
                    // notice we return the original result not the promise with the logging call.
                    if (typeof promise.catch === "function") {
                        promise.catch((e: any) => e);
                    }
                } else {
                    exitLog();
                }

                return result;
            };

            Object.defineProperty(target.prototype, propertyName, descriptor);
        }
    };
}
```

I was hoping that there would be a better way of detecting an async function but at this time there does not seem to be.

Keep in mind in production we use a singleton service for the logging instead of direct `console.log` calls which also add the timestamp and the log level in the examples.

The decorator can be applied to any class like so:

```typescript
import { Loggable } from "<path>/loggable.decorator";

@Loggable()
export class AuthService {
    async login(...) {...}
    async getUserForLogin(...) {...}
    async createToken(...) {...}
}
```

And then you will start to see logs like:

```bash
2018-12-29T13:51:36.984Z debug: [AuthService][login]	Entering
2018-12-29T13:51:36.985Z debug: [AuthService][getUserForLogin]	Entering
2018-12-29T13:51:37.079Z debug: [AuthService][getUserForLogin]	Exiting 94ms
2018-12-29T13:51:37.079Z debug: [AuthService][createToken]	Entering
2018-12-29T13:51:37.080Z debug: [AuthService][createToken]	Exiting 1ms
2018-12-29T13:51:37.092Z debug: [AuthService][login]	Exiting 108ms
```

Also keep in mind the decorator is applied when the class is declared. In node, [under most circumstances](https://medium.com/@lazlojuly/are-node-js-modules-singletons-764ae97519af), this happens the first time the modules is imported.
