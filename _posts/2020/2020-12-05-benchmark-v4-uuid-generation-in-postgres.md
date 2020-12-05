# Benchmark v4 uuid generation in postgres

**05/12/2020**

TLDR: use `gen_random_uuid` to generate v4 uuids.

There are two main functions to generate v4 uuids in postgres, `uuid_generate_v4` and `gen_random_uuid`. In postgres 13, `gen_random_uuid` is a built in function. Otherwise you will need to install the `pgcrypto` extension. `uuid_generate_v4` requires the `uuid-ossp` extension. Depending how postgres is configured, postgres may actually use different libraries for the `uuid-ossp` extension (ossp-uuid, libc, libuuid) see [postgres doc](https://www.postgresql.org/docs/13/uuid-ossp.html) for more info.

## Tests

```sql
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pgcrypto";
```

```sql
EXPLAIN ANALYZE SELECT uuid_generate_v4() FROM generate_series(1, 10000);
EXPLAIN ANALYZE SELECT gen_random_uuid() FROM generate_series(1, 10000);
```

## Notes

- All benchmarks used postgres 12.x.
- On windows, postgres was installed using https://www.enterprisedb.com.
- On linux, postgres was installed using respective package managers.
- All tests were done on google cloud VMs.
  - e2-medium: (2 vCPUs, 4 GB, HDD)
  - e2-standard-4: (4 vCPUs, 16 GB, HDD)

## Results

| hardware      | platform            | uuid_generate_v4 | gen_random_uuid |
| ------------- | ------------------- | ---------------- | --------------- |
| e2-medium     | ubuntu-16.04        | 95ms             | 10ms            |
| e2-standard-4 | centos-8            | 80ms             | 30ms            |
| e2-medium     | windows-server-2012 | 1800ms           | 5ms             |
| e2-medium     | windows-server-2016 | 3600ms           | 5ms             |
| e2-standard-4 | windows-server-2019 | 4400ms           | 5ms             |

More investigation is required to determine why `uuid_generate_v4()` is so slow on windows server. It's unclear what underlying lib the enterprisedb postgres is using for `uuid-ossp`. Regardless of why it's slow on windows server, the best option for now is to use `gen_random_uuid` which is significantly faster on all platforms tested and comes built in on postgres 13.

**update**

To be sure the windows issue was not limited to the edb postgres 12.4 version, I did one quick benchmark on the latest edb postgres 13 and found similar execution times for uuid_generate_v4.
