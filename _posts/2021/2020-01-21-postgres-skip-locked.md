# Skip locked in postgres

**21/01/2021**

When doing an update query, postgres will lock each row it finds until the transaction is committed or rolled back. For most use cases this is desired behavior, it means concurrent updates will be persisted correctly or throw a deadlock error. Using [SKIP LOCKED](https://www.postgresql.org/docs/12/sql-select.html) in combination with an update query allows us to skip any further updates while any row is locked. This is useful for clients that send a burst of updates in a short amount of time but only care that the first update is persisted.

Example:

```sql
UPDATE "item"
SET "updated_at" = CURRENT_TIMESTAMP
WHERE "item"."group_id" =
    (
        SELECT "id"
        FROM "group"
        WHERE "group"."name" = $1
        ORDER BY "id"
        FOR UPDATE SKIP LOCKED
        LIMIT 1
    );
```
