# Alternative to array comparisons in postgres

__20/04/2020__

![translations](https://imgs.xkcd.com/comics/mistranslations.png)

[Array comparisons](https://www.postgresql.org/docs/current/functions-comparisons.html) make comparisons between groups of values easy:

```sql
DELETE FROM user WHERE name IN ('fred', 'bean');
```

In a general purpose language like typescript, the filter fragment is often considered to be equivalent to:

```typescript
const where = ['fred', 'bean'].includes(name);
```

However array comparisons in postgres do not handle emptyness. e.g:

```sql
DELETE FROM user WHERE name IN (); // sql error
```

Which often leads to various bugs when building SQL queries from a GPL. e.g https://github.com/typeorm/typeorm/issues/2195.

There are various workarounds, the most common is to handle the emptyness at the application level:

```typescript
if (safeInput.length) {
    const query = `DELETE FROM user WHERE name IN (${ safeInput.join(", ") });`
}
```

which works for most cases but requires some special logic for the negation query:

```sql
DELETE FROM user WHERE name NOT IN ('fred', 'bean');
```

An alternative way of achieving the same result is to use [array functions](https://www.postgresql.org/docs/current/functions-array.html).
Array functions handle empty arrays and have similar semantics to array functions in a GPL.

```sql
DELETE FROM user WHERE array[name] && array['fred', 'bean]::text[];
```

The drawback here is postgres cannot determine a type of an empty array, which means we have to explicitly cast to the correct array type.
But I like it better than using an array comparison because it will handle the negation as well. e.g:

```sql
DELETE FROM user WHERE NOT (array[name] && array['fred', 'bean]::text[]);
```

Full sql example below:

```sql
CREATE TABLE IF NOT EXISTS "user" (name text PRIMARY KEY);

INSERT INTO "user" ("name") VALUES ('c') ON CONFLICT (name) DO NOTHING;

SELECT * FROM "user";

DELETE FROM "user" WHERE NOT (array[name] && array['c']::text[]);

DELETE FROM "user" WHERE name NOT IN ('c');
```
