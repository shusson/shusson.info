# Investigating memory leaks in e2e tests with jest

__23/08/2018__

![decorator](/assets/manual_gc_no_fixes.png)

#### Why

- could no longer run e2e tests
- memory consumption reached 1GB for 40 tests

#### what

- https://github.com/CTcue/ctReach/pull/299
- node.js
  - expose-gc
- jest
  --detectLeaks and --logHeapUsage
- chrome debugging tools

#### how

- create small isolated tests and run multiple times but make sure you enable `expose-gc
- use --detectLeaks
- use chrome memory profiler

Detected leaks:
- niftylettuce/email-templates#318
- TooTallNate/node-agent-base#22
- winstonjs/winston#1445

- closing elasticsearch connection
