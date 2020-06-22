# Generating test data in postgres

__04/06/2020__

![data](https://imgs.xkcd.com/comics/data.png)

Using the [generate_series](https://www.postgresql.org/docs/12/functions-srf.html) postgres function
we can quickly generate large datasets to test various scenarios for a given schema.

First, connect postgres, create a test db, enable timing and increase the statement_timeout.

```bash
psql -U postgres
postgres=# CREATE DATABASE test;
CREATE DATABASE
postgres=# \c test;
You are now connected to database "test" as user "postgres".
test=# \timing on
Timing is on.
test=# SET statement_timeout = 100000;
SET
```

Now define a simple schema.

```sql
DROP TABLE IF EXISTS search_result;
DROP TABLE IF EXISTS study;

CREATE TABLE study (
    id text primary key,
    description text
);

CREATE TABLE search_result (
    id serial primary key,
    name text,
    study_id text references study(id)
);
```

The `study` table represents a research study in a hospital, e.g `Obesity rates of patients with hypertension`.
The `search_result` table represents the patient information found for a particular study.

Each `study` can have many `search_results`.

Let's generate some `search_results` for this schema.

Manually add a few studies:
```sql
INSERT INTO
    study (id, description)
VALUES
    (
        'OBESITY',
        'Lorem ipsum dolor sit amet, consectetur adipiscing elit'
    ),
   	(
        'DIABETES',
        'Lorem ipsum dolor sit amet, consectetur adipiscing elit'
    ),
   	(
        'HYPERTENSION',
        'Lorem ipsum dolor sit amet, consectetur adipiscing elit'
    );
```

Generate a hundred thousand results:
```sql
INSERT INTO
    search_result (name, study_id)
SELECT
    md5(random() :: text),
    'OBESITY'
FROM
    generate_series(1, 100000) s(i);
```

```
INSERT 0 100000
Time: 902.766 ms
```

Generate a further million:
```sql
INSERT INTO
    search_result (name, study_id)
SELECT
    md5(random() :: text),
    'HYPERTENSION'
FROM
    generate_series(1, 1000000) s(i);
```

```
INSERT 0 1000000
Time: 9142.263 ms (00:09.142)
```

Generate a further 10 million results (depending on machine specs, you may have to increase the sessions `statement_timeout`):
```sql
INSERT INTO
    search_result (name, study_id)
SELECT
    md5(random() :: text),
    'DIABETES'
FROM
    generate_series(1, 10000000) s(i);
```

```
INSERT 0 10000000
Time: 91913.053 ms (01:31.913)
```

To make sure testing results are consistent it's always good to do an manual vacuum and analyze before testing.
In most cases the `auto-daemon` should kick in, but it depends on the postgres configuration and the order in which you generated your test data
(the auto-daemon runs when x percent of the rows of the table have changed).

```sql
VACUUM (ANALYZE);
```

You can verify the table has been vacuumed with:
```sql
SELECT relname, n_live_tup, n_dead_tup, last_autoanalyze, last_autovacuum, last_analyze, last_vacuum FROM pg_stat_all_tables where "schemaname"='public' ORDER BY last_vacuum DESC NULLS LAST;
```

```
    relname    | n_live_tup | n_dead_tup |       last_autoanalyze        | last_autovacuum |         last_analyze          |          last_vacuum
---------------+------------+------------+-------------------------------+-----------------+-------------------------------+-------------------------------
 search_result |   11103406 |          0 | 2020-06-13 10:07:43.560753+00 |                 | 2020-06-13 10:16:00.730518+00 | 2020-06-13 10:16:00.434277+00
 study         |          3 |          0 |                               |                 | 2020-06-13 10:16:00.430135+00 | 2020-06-13 10:16:00.42881+00
```

It seems the mismatch between `n_live_tup` and `row count` is a [known issue](https://postgrespro.com/list/thread-id/1520106).

Now we can test the performance of some simple queries against the data.
For example we can investigate the cost/benefit of adding an index to the `study_id` foreign key (postgres does not add indexes to foreign keys by default).


```sql
SELECT COUNT(*) FROM search_result WHERE study_id = 'OBESITY';
```

```
 count
--------
 100000
(1 row)


Time: 518.761 ms
```

Add an index:

```sql
CREATE INDEX search_result_study_id_idx ON search_result(study_id);
```

```
CREATE INDEX
Time: 6369.998 ms (00:06.370)
```

Since our index is on a simple column, it is [not necessary](https://dba.stackexchange.com/questions/241257/is-it-necessary-to-analyze-a-table-after-an-index-has-been-created#:~:text=If%20the%20index%20is%20just,after%20you%20create%20the%20index.&text=However%2C%20if%20you%20are%20indexing,ANALYZE%20after%20creating%20the%20index.) to ANALYZE the table again. However more complex indexes may require an ANALYZE.


The new cost of the select:
```
 SELECT COUNT(*) FROM search_result WHERE study_id = 'OBESITY';
 count
--------
 100000
(1 row)


Time: 13.148 ms
```

We can also investigate the new cost of inserting the data now that we have the index.

Be careful if you use `DELETE` instead of `TRUNCATE` because `DELETE` may require a vacuum and analyze (depending on configuration).

```text
test=# TRUNCATE search_result;
TRUNCATE TABLE
Time: 87.748 ms
test=# INSERT INTO
test-#     search_result (name, study_id)
test-# SELECT
test-#     md5(random() :: text),
test-#     'OBESITY'
test-# FROM
test-#     generate_series(1, 100000) s(i);
INSERT 0 100000
Time: 1140.551 ms (00:01.141)
test=# INSERT INTO
test-#     search_result (name, study_id)
test-# SELECT
test-#     md5(random() :: text),
test-#     'HYPERTENSION'
test-# FROM
test-#     generate_series(1, 1000000) s(i);
INSERT 0 1000000
Time: 13149.848 ms (00:13.150)
test=# INSERT INTO
test-#     search_result (name, study_id)
test-# SELECT
test-#     md5(random() :: text),
test-#     'DIABETES'
test-# FROM
test-#     generate_series(1, 10000000) s(i);
INSERT 0 10000000
Time: 131413.735 ms (02:11.414)
```

Machine specs:

    - AMD Ryzen 5 3600
    - NVME 512GB SAMSUNG SSD 970 EVO Plus
    - 32GB DDR4
