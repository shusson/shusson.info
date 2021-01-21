# Collecting query statistics with postgres

**30/10/2020**

Getting basic insights into query performance is easy with the `pg_stat_statements` module. For applications that use postgres, being able to see min/max/avg query times can help track down non-performing queries that might otherwise be missed by other means. Keep in mind there is a [performance impact](http://pgsnaga.blogspot.com/2011/10/performance-impact-of-pgstatstatements.html) of enabling `pg_stat_statements`.

Add `pg_stat_statements` to the `shared_preload_libraries` in postgres.conf

```
shared_preload_libraries = 'pg_stat_statements'
```

Find out where your postgres file is:

```sql
SHOW config_file;

-- /usr/local/var/postgres/postgresql.conf
```

Restart postgres.

Connect to a specific database and create the extension.

```sql
CREATE EXTENSION pg_stat_statements;
```

Once enabled postgres will start collecting statistics for sql statements.

Some useful queries:

```sql
SELECT
    calls,
    ROUND(total_time::numeric, 2) as total,
    ROUND(max_time::numeric, 2) as max,
    ROUND(mean_time::numeric, 2) as mean,
    left(query, 50) as query
FROM
    pg_stat_statements
ORDER BY
    max_time DESC
LIMIT
    10;
```

```sql
SELECT
    calls,
    ROUND(total_time::numeric, 2) as total,
    ROUND(max_time::numeric, 2) as max,
    ROUND(mean_time::numeric, 2) as mean,
    left(query, 50) as query
FROM
    pg_stat_statements
ORDER BY
    total_time DESC
LIMIT
    10;
```

Example output:

```
calls|total  |max    |mean   |query                                             |
-----|-------|-------|-------|--------------------------------------------------|
    3|3289.59|1289.06|1096.53|INSERT INTO¶    search_result (name, study_id)¶SEL|
    1| 690.77| 690.77| 690.77|VACUUM (ANALYZE)                                  |
    1| 132.38| 132.38| 132.38|CREATE INDEX search_result_study_id_idx ON search_|
    3| 118.68|  41.03|  39.56|SELECT COUNT(*) FROM search_result WHERE study_id |
    1|  47.87|  47.87|  47.87|SELECT relname, n_live_tup, n_dead_tup, last_autoa|
    1|   9.37|   9.37|   9.37|CREATE EXTENSION pg_stat_statements               |
    1|   9.37|   9.37|   9.37|CREATE TABLE search_result (¶    id serial primary|
    1|   5.57|   5.57|   5.57|SELECT t.oid,t.*,c.relkind,format_type(nullif(t.ty|
    1|   3.95|   3.95|   3.95|CREATE TABLE study (¶    id text primary key,¶    |
    2|   3.02|   1.73|   1.51|SELECT x.oid,x.* FROM pg_catalog.pg_proc x WHERE x|
```

To reset the statistics:

```
pg_stat_statements_reset();
```
