# Postgres performance of upsert vs update

__05/11/2020__

upsert
```sql
EXPLAIN ANALYZE INSERT INTO public.search_result (id,"name",study_id) VALUES
	 (1,'111e768ace70b3782e55e85b773e8988','OBESITY'),
     ...
     (10000,'3d65650a44981a65a45092b99f44bd22','OBESITY')
     ON CONFLICT (id) DO UPDATE
	 SET name = excluded.name;
```

```
Insert on search_result  (cost=0.00..125.00 rows=10000 width=68) (actual time=74.070..74.070 rows=0 loops=1)
  Conflict Resolution: UPDATE
  Conflict Arbiter Indexes: search_result_pkey
  Tuples Inserted: 0
  Conflicting Tuples: 10000
  ->  Values Scan on "*VALUES*"  (cost=0.00..125.00 rows=10000 width=68) (actual time=0.002..6.068 rows=10000 loops=1)
Planning Time: 2.561 ms
Execution Time: 74.299 ms
```

Update from with values

```sql
EXPLAIN ANALYZE UPDATE search_result SET
        name = input.name
    FROM (
        VALUES
        (1,'111e768ace70b3782e55e85b773e8988','OBESITY'),
        ...
        (10000,'3d65650a44981a65a45092b99f44bd22','OBESITY')
    ) AS input("id", "name", "study_id")
    WHERE search_result.id = input.id;
```

```
Update on search_result  (cost=250.00..7068.43 rows=10000 width=110) (actual time=50.445..50.447 rows=0 loops=1)
  ->  Hash Join  (cost=250.00..7068.43 rows=10000 width=110) (actual time=7.356..14.716 rows=10000 loops=1)
        Hash Cond: (search_result.id = "*VALUES*".column1)
        ->  Seq Scan on search_result  (cost=0.00..5630.13 rows=290213 width=18) (actual time=0.122..4.175 rows=10000 loops=1)
        ->  Hash  (cost=125.00..125.00 rows=10000 width=96) (actual time=7.219..7.220 rows=10000 loops=1)
              Buckets: 16384  Batches: 1  Memory Usage: 1457kB
              ->  Values Scan on "*VALUES*"  (cost=0.00..125.00 rows=10000 width=96) (actual time=0.005..4.892 rows=10000 loops=1)
Planning Time: 4.723 ms
Execution Time: 51.434 ms
```


## Test data

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

INSERT INTO
    study (id, description)
VALUES
    (
        'OBESITY',
        'Lorem ipsum dolor sit amet, consectetur adipiscing elit'
    );


INSERT INTO
    search_result (name, study_id)
SELECT
    md5(random() :: text),
    'OBESITY'
FROM
    generate_series(1, 10000) s(i);
```
