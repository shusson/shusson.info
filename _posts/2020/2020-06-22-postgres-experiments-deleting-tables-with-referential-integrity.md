# Postgres experiments: deleting tables with referential integrity

__22/06/2020__

![referential_integrity](https://imgs.xkcd.com/comics/bag_check.png)

Adding a foreign key constraint in postgres adds significant cost to deletions.

Here is a common "one-to-many" relationship, where each row in the `large_table` can have a link to many rows in the `small_table`.

```sql
DROP TABLE IF EXISTS large_table CASCADE;
DROP TABLE IF EXISTS small_table CASCADE;

CREATE TABLE large_table (
    id serial primary key,
    name text
);

CREATE TABLE small_table (
    id serial primary key,
    large_table_id serial REFERENCES large_table(id) ON DELETE CASCADE
);
```

Insert some test data:
```sql
INSERT INTO
    large_table (name)
SELECT
    md5(random() :: text)
FROM
    generate_series(1, 100000) s(i);
```

Delete rows from the large table:
```sql
-- (in reality we'll be deleting a subset of rows and using an index)
EXPLAIN ANALYZE DELETE FROM "large_table";
```

```
Delete on large_table  (cost=0.00..1893.18 rows=105918 width=6) (actual time=88.630..88.630 rows=0 loops=1)
  ->  Seq Scan on large_table  (cost=0.00..1893.18 rows=105918 width=6) (actual time=0.012..11.345 rows=100000 loops=1)
Planning Time: 0.089 ms
Trigger for constraint small_table_large_table_id_fkey: time=477.615 calls=100000
Execution Time: 570.642 ms
```

Even though there is no data in the `small_table`, postgres is spending ~80% checking the FK constraint.

```
| large_table size (rows) | total time (ms) |
| -                       | -               |
| 10000                   | 50              |
| 100000                  | 500             |
| 1000000                 | 5000            |
```

Adding more tables with referencing keys increases the cost linearly:

```
| large_table size (rows) | number of referencing tables | total time (ms) |
| -                       | -                            | -               |
| 100000                  | 1                            | 500             |
| 100000                  | 2                            | 1000            |
| 100000                  | 3                            | 1500            |
```

Changing the constraint to `ON DELETE NO ACTION` actually increases response times:

```sql
CREATE TABLE small_table (
    id serial primary key,
    large_table_id serial REFERENCES large_table(id) ON DELETE NO ACTION
);
```

```
| large_table size (rows) | total time (ms) |
| -                       | -               |
| 10000                   | 100             |
| 100000                  | 1000            |
| 1000000                 | 10000           |
```

Now with some data in the `small_table`:

```sql
INSERT INTO
    small_table (large_table_id)
SELECT
    (SELECT id FROM large_table
	ORDER BY RANDOM()
	LIMIT 1)
FROM
    generate_series(1, 1000) s(i);

EXPLAIN ANALYZE DELETE FROM "large_table";
```

```
| large_table size (rows) | small_table size (rows) | total time (ms) |
| -                       | -                       | -               |
| 10000                   | 1000                    | 500             |
| 100000                  | 1000                    | 4000            |
| 1000000                 | 1000                    | 60000           |
```

We can significantly improve the response time by adding an index to the FK in the small table:

```sql
CREATE INDEX large_table_id_idx ON small_table(large_table_id);
```

```
| large_table size (rows) | small_table size (rows) | total time (ms) |
| -                       | -                       | -               |
| 10000                   | 1000                    | 75              |
| 100000                  | 1000                    | 750             |
| 1000000                 | 1000                    | 7500            |
```

If you really must, there are options to [disable the FK triggers temporarily](https://stackoverflow.com/questions/62452621/how-to-optimize-deleting-rows-from-a-large-table-which-has-referential-integrity).

Machine specs:

    - AMD Ryzen 5 3600
    - NVME 512GB SAMSUNG SSD 970 EVO Plus
    - 32GB DDR4


