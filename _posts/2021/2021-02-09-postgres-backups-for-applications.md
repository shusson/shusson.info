# Postgres backups for applications

**09/02/2021**

Postgres provides two tools for backing up a single database, [pg_dump](https://www.postgresql.org/docs/12/app-pgdump.html) and [pg_restore](https://www.postgresql.org/docs/12/app-pgrestore.html). `pg_dump` is used for backing up a single databases and `pg_restore` is used for restoring backups made by `pg_dump`. There are many options for these tools, but this post is focused on backing up and restoring the database exactly as it was, which is often the case when restoring an application database.

Because the default options require admin rights you will most likely always want to connect as a superuser (e.g postgres). It goes without saying, but do not run these commands without first testing them on test databases.

## pg_dump and pg_restore

There are two main output formats for `pg_dump`, sql-scripts or archive file formats. This post will focus on the archive formats since the sql-scripts require a lot of options upfront. For example `--no-owner` or `--create`, will need to specified when you create the dump, which makes the dump a lot less flexible.

```bash
PGUSER=postgres PGPASSWORD=<pwd> pg_dump -Fc <database> <path-to-backup>
```

`-Fc` means we use pg_dump's custom format which is also compressed by default.

Then to restore the backup file:

```bash
PGUSER=postgres PGPASSWORD=xxx pg_restore -C -d postgres <path-to-backup>
```

`-C` tells `pg_restore` to create the database in the dump. It requires that there is no existing database with the same name. If you want to restore into a different database than the one in the dump, rename the db after `pg_restore` has run.

`-d postgres` is the database where pg_restore will apply commands. If you don't use `-C`, `pg_restore` will restore into this database (bad).

## Extra notes

If you are restoring but there's already an existing database, then you can rename the db with:

```sql
ALTER DATABASE <db> RENAME TO <new_db>;
```

Example of using `pg_dump` with sql-script format

```bash
PGUSER=postgres pg_dump old > db.sql
PGUSER=postgres psql -d new -f db.sql
```

### pg_dumpall

[pg_dumpall](https://www.postgresql.org/docs/12/app-pg-dumpall.html) is similar to `pg_dump` but will backup all databases at once. However it does not support the archive file format which means options need to be specified up-front and you cannot use the output with `pg_restore`

```bash
PGUSER=postgres pg_dumpall > db.out
PGUSER=postgres psql -f db.out postgres
```
