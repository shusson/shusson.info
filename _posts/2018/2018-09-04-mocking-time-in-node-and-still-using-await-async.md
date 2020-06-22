# Mocking time in node and still using await/async

__04/09/2018__

![decorator](https://static1.squarespace.com/static/55ef0e29e4b099e22cdc9eea/t/57a4757d893fc0b30a4a53d7/1470395779879/?format=1500w)

Sometimes you need to run an end-to-end test that depends on the native clock. For example a job service might dequeue a job to be processed every 5 seconds. We want to be able to test some behavior which depends on the job service but we don't want to have to wait 5 seconds. In node, and other javascript environments, we can do this by mocking the internal clock using a library called [lolex](https://github.com/sinonjs/lolex). Lolex supports automatically incrementing time which makes it easy to use the async/await functions.

```javascript
// replaces the native timers with the fake one.
// the option `shouldAdvanceTime` allows us to use the await
const clock = lolex.install({ shouldAdvanceTime: true, advanceTimeDelta: 1 });

// start an export job
const res = await request(app.listen())
    .post(`/export/1`)
    .set("authorization", `Bearer ${token}`)
    .expect(200);

// move the clock forward by 6 seconds
clock.tick(6000);

// verify the job is complete.
...
```
