---
title: Disk Usage
source: https://wiki.postgresql.org/wiki/Disk_Usage
kind: external
domain: database
author: PostgreSQL Project
license: postgresql-license
fetched_at: 2026-05-18
tags: [external, database]
---

> [!info] External article · imported reference
> Source: [wiki.postgresql.org](https://wiki.postgresql.org/wiki/Disk_Usage)
> Author: PostgreSQL Project
> License: postgresql-license
> Fetched: 2026-05-18

# Disk Usage

## Finding the size of various object in your database

## General Table Size Information Grouped For Partitioned Tables

This will report size information for all tables, that are not inherited, in the "pretty" form. Inherited tables are grouped together.

```
WITH RECURSIVE pg_inherit(inhrelid, inhparent) AS
    (select inhrelid, inhparent
    FROM pg_inherits
    UNION
    SELECT child.inhrelid, parent.inhparent
    FROM pg_inherit child, pg_inherits parent
    WHERE child.inhparent = parent.inhrelid),
pg_inherit_short AS (SELECT * FROM pg_inherit WHERE inhparent NOT IN (SELECT inhrelid FROM pg_inherit))
SELECT table_schema
    , TABLE_NAME
    , row_estimate
    , pg_size_pretty(total_bytes) AS total
    , pg_size_pretty(index_bytes) AS INDEX
    , pg_size_pretty(toast_bytes) AS toast
    , pg_size_pretty(table_bytes) AS TABLE
    , total_bytes::float8 / sum(total_bytes) OVER () AS total_size_share
  FROM (
    SELECT *, total_bytes-index_bytes-COALESCE(toast_bytes,0) AS table_bytes
    FROM (
         SELECT c.oid
              , nspname AS table_schema
              , relname AS TABLE_NAME
              , SUM(c.reltuples) OVER (partition BY parent) AS row_estimate
              , SUM(pg_total_relation_size(c.oid)) OVER (partition BY parent) AS total_bytes
              , SUM(pg_indexes_size(c.oid)) OVER (partition BY parent) AS index_bytes
              , SUM(pg_total_relation_size(reltoastrelid)) OVER (partition BY parent) AS toast_bytes
              , parent
          FROM (
                SELECT pg_class.oid
                    , reltuples
                    , relname
                    , relnamespace
                    , pg_class.reltoastrelid
                    , COALESCE(inhparent, pg_class.oid) parent
                FROM pg_class
                    LEFT JOIN pg_inherit_short ON inhrelid = oid
                WHERE relkind IN ('r', 'p')
             ) c
             LEFT JOIN pg_namespace n ON n.oid = c.relnamespace
  ) a
  WHERE oid = parent
) a
ORDER BY total_bytes DESC;
```

## General Table Size Information Grouped For Partitioned Tables

Will show tables like above, but sizes split individually for each tablespace.

```
WITH RECURSIVE pg_inherit(inhrelid, inhparent) AS
    (select inhrelid, inhparent
    FROM pg_inherits
    UNION
    SELECT child.inhrelid, parent.inhparent
    FROM pg_inherit child, pg_inherits parent
    WHERE child.inhparent = parent.inhrelid),
pg_inherit_short AS (SELECT * FROM pg_inherit WHERE inhparent NOT IN (SELECT inhrelid FROM pg_inherit))
SELECT parent::regclass
    , coalesce(spcname, 'default') pg_tablespace_name
    , row_estimate
    , pg_size_pretty(total_bytes) AS total
    , pg_size_pretty(index_bytes) AS INDEX
    , pg_size_pretty(toast_bytes) AS toast
    , pg_size_pretty(table_bytes) AS TABLE
    , 100 * total_bytes::float8 / sum(total_bytes) OVER () AS PERCENT
  FROM (
    SELECT *, total_bytes-index_bytes-COALESCE(toast_bytes,0) AS table_bytes
    FROM (
         SELECT parent
              , reltablespace
              , SUM(c.reltuples) AS row_estimate
              , SUM(pg_total_relation_size(c.oid)) AS total_bytes
              , SUM(pg_indexes_size(c.oid)) AS index_bytes
              , SUM(pg_total_relation_size(reltoastrelid)) AS toast_bytes
          FROM (
                SELECT pg_class.oid
                    , reltuples
                    , relname
                    , relnamespace
                    , reltablespace reltablespace
                    , pg_class.reltoastrelid
                    , COALESCE(inhparent, pg_class.oid) parent
                FROM pg_class
                    LEFT JOIN pg_inherit_short ON inhrelid = oid
                WHERE relkind IN ('r', 'p')
             ) c
            GROUP BY parent, reltablespace
  ) a
) a LEFT JOIN pg_tablespace ON (pg_tablespace.oid = reltablespace)
ORDER BY total_bytes DESC;
```

## General Table Size Information

This will report size information for all tables, in both raw bytes and "pretty" form.

```
SELECT *, pg_size_pretty(total_bytes) AS total
    , pg_size_pretty(index_bytes) AS index
    , pg_size_pretty(toast_bytes) AS toast
    , pg_size_pretty(table_bytes) AS table
  FROM (
  SELECT *, total_bytes-index_bytes-coalesce(toast_bytes,0) AS table_bytes FROM (
      SELECT c.oid,nspname AS table_schema, relname AS table_name
              , c.reltuples AS row_estimate
              , pg_total_relation_size(c.oid) AS total_bytes
              , pg_indexes_size(c.oid) AS index_bytes
              , pg_total_relation_size(reltoastrelid) AS toast_bytes
          FROM pg_class c
          LEFT JOIN pg_namespace n ON n.oid = c.relnamespace
          WHERE relkind = 'r'
  ) a
) a;
```

## Finding the largest databases in your cluster

Databases to which the user cannot connect are sorted as if they were infinite size.

```
SELECT d.datname as Name,  pg_catalog.pg_get_userbyid(d.datdba) as Owner,
    CASE WHEN pg_catalog.has_database_privilege(d.datname, 'CONNECT')
        THEN pg_catalog.pg_size_pretty(pg_catalog.pg_database_size(d.datname))
        ELSE 'No Access'
    END as Size
FROM pg_catalog.pg_database d
    order by
    CASE WHEN pg_catalog.has_database_privilege(d.datname, 'CONNECT')
        THEN pg_catalog.pg_database_size(d.datname)
        ELSE NULL
    END desc -- nulls first
    LIMIT 20;
```

## Finding the size of your biggest relations

Relations are objects in the database such as tables and indexes, and this query shows the size of all the individual parts. Tables which have both regular and [TOAST](http://www.postgresql.org/docs/current/static/storage-toast.html) pieces will be broken out into separate components; an example showing how you might include those into the main total is available in the [documentation](http://www.postgresql.org/docs/current/static/disk-usage.html), and as of PostgreSQL 9.0 it's possible to include it automatically by using pg_table_size here instead of pg_relation_size:

Note that all of the queries below this point on this page show you the sizes for only those objects which are in the database you are currently connected to.

```
SELECT nspname || '.' || relname AS "relation",
    pg_size_pretty(pg_relation_size(C.oid)) AS "size"
  FROM pg_class C
  LEFT JOIN pg_namespace N ON (N.oid = C.relnamespace)
  WHERE nspname NOT IN ('pg_catalog', 'information_schema')
  ORDER BY pg_relation_size(C.oid) DESC
  LIMIT 20;
```

Example output (from a database created with pgbench, scale=25):

```` ```
        relation        |    size    
------------------------+------------
 public.accounts        | 326 MB
 public.accounts_pkey   | 44 MB
 public.history         | 592 kB
 public.tellers_pkey    | 16 kB
 public.branches_pkey   | 16 kB
 public.tellers         | 16 kB
 public.branches        | 8192 bytes
``` ````

## Finding the total size of your biggest tables

This version of the query uses [pg_total_relation_size](http://www.postgresql.org/docs/current/static/functions-admin.html#FUNCTIONS-ADMIN-DBSIZE), which sums total disk space used by the table including indexes and toasted data rather than breaking out the individual pieces:

```
SELECT nspname || '.' || relname AS "relation",
    pg_size_pretty(pg_total_relation_size(C.oid)) AS "total_size"
  FROM pg_class C
  LEFT JOIN pg_namespace N ON (N.oid = C.relnamespace)
  WHERE nspname NOT IN ('pg_catalog', 'information_schema')
    AND C.relkind <> 'i'
    AND nspname !~ '^pg_toast'
  ORDER BY pg_total_relation_size(C.oid) DESC
  LIMIT 20;
```

## Easy access to these queries

[~/.psqlrc tricks: table sizes](https://github.com/datachomp/dotfiles/blob/master/.psqlrc#L53) shows how to make it easy to run size related queries like this in psql.
