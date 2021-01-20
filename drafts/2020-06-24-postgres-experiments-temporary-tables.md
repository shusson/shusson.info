# Postgres experiments: Temporary tables

__24/06/2020__

!(tmp)[tmp]

#### Why

- Use postgres to process some data
- Sacrifice redundancy for speed

# what

- create table with the same schema, compare temp vs regular on simple inserts and queries

The regular table:

```sql
DROP TABLE IF EXISTS report;
CREATE TABLE report (
    id serial PRIMARY KEY,
    start_date TIMESTAMP WITH TIME ZONE,
    content TEXT
);

INSERT INTO
    report (start_date, content)
SELECT
   x,
   md5(random() :: text)
FROM
     generate_series(timestamp '2000-01-01 00:00'
                     , timestamp '2020-01-01 00:00'
                     , interval  '10 min') t(x);


```

```sql
BEGIN;


CREATE TEMP TABLE report (
    id serial PRIMARY KEY,
    start_date TIMESTAMP WITH TIME ZONE,
    content TEXT
) ON COMMIT DROP;


INSERT INTO
    report (start_date, content)
SELECT
   x,
   md5(random() :: text)
FROM
     generate_series(timestamp '2000-01-01 00:00'
                     , timestamp '2020-01-01 00:00'
                     , interval  '10 min') t(x);
SELECT COUNT(*) FROM report LIMIT 10;
COMMIT;

```

# how


