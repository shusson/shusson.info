# Dealing with deadlocks in postgres

__25/11/2019__

![deadlock](https://imgs.xkcd.com/comics/dependencies.png)

Recently we started running into deadlocks in our application. The first thing to do is to look at the postgres logs.

```bash
2019-11-14 15:24:23.326 UTC [1] ERROR:  deadlock detected
2019-11-14 15:24:23.326 UTC [1] DETAIL:  Process 1 waits for ShareLock on transaction 198234; blocked by process 2.
	Process 2 waits for ShareLock on transaction 198232; blocked by process 1.
	Process 1: UPDATE "query-result-cache" SET "identifier" = $1 WHERE "query" = $6
	Process 2: UPDATE "query-result-cache" SET "identifier" = $1 WHERE "query" = $6
2019-11-14 15:24:23.326 UTC [1] HINT:  See server log for query details.
2019-11-14 15:24:23.326 UTC [1] CONTEXT:  while locking tuple (7,7) in relation "query-result-cache"
2019-11-14 15:24:23.326 UTC [1] STATEMENT:  UPDATE "query-result-cache" SET "identifier" = $1 WHERE "query" = $6
```

Postgres is telling us that process 1 is blocked by process 2 and process 2 is blocked by process 1. Postgres can get into this state if two transactions
concurrently modify a table. While transactions are running, postgres will lock rows, which under certain scenarios leads to deadlock. The simplest example is

Transaction 1 (T1):

```sql
UPDATE "query-result-cache" SET "identifier" = $1 WHERE "query" = 'X';
UPDATE "query-result-cache" SET "identifier" = $1 WHERE "query" = 'Y';
```

Transaction 2 (T2):

```sql
UPDATE "query-result-cache" SET "identifier" = $1 WHERE "query" = 'Y';
UPDATE "query-result-cache" SET "identifier" = $1 WHERE "query" = 'X';
```

Execution sequence:

```bash
// T1 updates row X and acquires a lock.
UPDATE "query-result-cache" SET "identifier" = $1 WHERE "query" = 'X';
// T2 updates row Y and acquires a lock.
UPDATE "query-result-cache" SET "identifier" = $1 WHERE "query" = 'Y';

// T1 tries to access row Y, but its locked by T2. So T1 waits.
// T2 tries to access row X, but its locked by T1. So T2 waits.
// deadlock
```

postgres documentation recommends avoiding these kinds of deadlocks by ensuring transactions acquire locks in a consistent order.

For example:

Transaction 1 (T1):

```sql
UPDATE "query-result-cache" SET "identifier" = $1 WHERE "query" = 'X';
UPDATE "query-result-cache" SET "identifier" = $1 WHERE "query" = 'Y';
```

Transaction 2 (T2):

```sql
UPDATE "query-result-cache" SET "identifier" = $1 WHERE "query" = 'X';
UPDATE "query-result-cache" SET "identifier" = $1 WHERE "query" = 'Y';
```

Potentially worse scenarios exist if the update affects multiple rows e.g:

```sql
UPDATE table_a SET x = 'value' WHERE id IN ($1, $2, $3)
```

As before postgres will lock rows in the update table and the locks will be held until the transaction commits or rolls back. BUT! Postgres updates rows in arbitrary order, postgres has no `ORDER BY` in the `UPDATE` command. With two concurrent update queries, postgres can end up in a deadlock in the same way that an application could cause postgres to deadlock. The way we can avoid deadlocks in this scenario is to tell postgres to explicitly lock the rows before the update.

```sql
UPDATE table_a t
SET x = 'value'
FROM (
   SELECT id
   FROM table_a
   WHERE id IN ($1, $2, $3)
   ORDER BY table_a.x
   FOR UPDATE
   ) cactus
WHERE t.id = cactus.id;
```

More info available in the [postgres doc](https://www.postgresql.org/docs/11/explicit-locking.html), through the [mailing list](https://www.postgresql.org/message-id/54EE24BC.4030402@pgmasters.net) and on [stackoverflow](https://stackoverflow.com/questions/27007196/avoiding-postgresql-deadlocks-when-performing-bulk-update-and-delete-operations)
