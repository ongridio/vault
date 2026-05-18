---
title: Lock Monitoring
source: https://wiki.postgresql.org/wiki/Lock_Monitoring
kind: external
domain: database
author: PostgreSQL Project
license: postgresql-license
fetched_at: 2026-05-18
tags: [external, database]
---

> [!info] External article · imported reference
> Source: [wiki.postgresql.org](https://wiki.postgresql.org/wiki/Lock_Monitoring)
> Author: PostgreSQL Project
> License: postgresql-license
> Fetched: 2026-05-18

# Lock Monitoring

# Online view current locks

## pg_locks view

Looking at [pg_locks](http://www.postgresql.org/docs/current/static/view-pg-locks.html) shows you what locks are granted and what processes are waiting for locks to be acquired. A good query to start looking for lock problems:

```
  select relation::regclass, * from pg_locks where not granted;
```

## pg_stat_activity view

- Figuring out what the processes holding or waiting for locks is easier if you cross-reference against the information in [pg_stat_activity](http://www.postgresql.org/docs/current/static/monitoring-stats.html)

## Сombination of blocked and blocking activity

The following query may be helpful to see what processes are blocking SQL statements (these only find row-level locks, not object-level locks).

```
  SELECT blocked_locks.pid     AS blocked_pid,
         blocked_activity.usename  AS blocked_user,
         blocking_locks.pid     AS blocking_pid,
         blocking_activity.usename AS blocking_user,
         blocked_activity.query    AS blocked_statement,
         blocking_activity.query   AS current_statement_in_blocking_process
   FROM  pg_catalog.pg_locks         blocked_locks
    JOIN pg_catalog.pg_stat_activity blocked_activity  ON blocked_activity.pid = blocked_locks.pid
    JOIN pg_catalog.pg_locks         blocking_locks 
        ON blocking_locks.locktype = blocked_locks.locktype
        AND blocking_locks.database IS NOT DISTINCT FROM blocked_locks.database
        AND blocking_locks.relation IS NOT DISTINCT FROM blocked_locks.relation
        AND blocking_locks.page IS NOT DISTINCT FROM blocked_locks.page
        AND blocking_locks.tuple IS NOT DISTINCT FROM blocked_locks.tuple
        AND blocking_locks.virtualxid IS NOT DISTINCT FROM blocked_locks.virtualxid
        AND blocking_locks.transactionid IS NOT DISTINCT FROM blocked_locks.transactionid
        AND blocking_locks.classid IS NOT DISTINCT FROM blocked_locks.classid
        AND blocking_locks.objid IS NOT DISTINCT FROM blocked_locks.objid
        AND blocking_locks.objsubid IS NOT DISTINCT FROM blocked_locks.objsubid
        AND blocking_locks.pid != blocked_locks.pid

    JOIN pg_catalog.pg_stat_activity blocking_activity ON blocking_activity.pid = blocking_locks.pid
   WHERE NOT blocked_locks.granted;
```

### Here's an alternate view of that same data that includes application_name's

Setting application_name variable in the begging of each transaction allows you to which logical process blocks another one. It can be information which source code line starts transaction or any other information that helps you to match application_name to your code.

```
SET application_name='%your_logical_name%';
```

```
SELECT blocked_locks.pid     AS blocked_pid,
         blocked_activity.usename  AS blocked_user,
         blocking_locks.pid     AS blocking_pid,
         blocking_activity.usename AS blocking_user,
         blocked_activity.query    AS blocked_statement,
         blocking_activity.query   AS current_statement_in_blocking_process,
         blocked_activity.application_name AS blocked_application,
         blocking_activity.application_name AS blocking_application
   FROM  pg_catalog.pg_locks         blocked_locks
    JOIN pg_catalog.pg_stat_activity blocked_activity  ON blocked_activity.pid = blocked_locks.pid
    JOIN pg_catalog.pg_locks         blocking_locks 
        ON blocking_locks.locktype = blocked_locks.locktype
        AND blocking_locks.DATABASE IS NOT DISTINCT FROM blocked_locks.DATABASE
        AND blocking_locks.relation IS NOT DISTINCT FROM blocked_locks.relation
        AND blocking_locks.page IS NOT DISTINCT FROM blocked_locks.page
        AND blocking_locks.tuple IS NOT DISTINCT FROM blocked_locks.tuple
        AND blocking_locks.virtualxid IS NOT DISTINCT FROM blocked_locks.virtualxid
        AND blocking_locks.transactionid IS NOT DISTINCT FROM blocked_locks.transactionid
        AND blocking_locks.classid IS NOT DISTINCT FROM blocked_locks.classid
        AND blocking_locks.objid IS NOT DISTINCT FROM blocked_locks.objid
        AND blocking_locks.objsubid IS NOT DISTINCT FROM blocked_locks.objsubid
        AND blocking_locks.pid != blocked_locks.pid
 
    JOIN pg_catalog.pg_stat_activity blocking_activity ON blocking_activity.pid = blocking_locks.pid
   WHERE NOT blocked_locks.GRANTED;
```

Note: While this query will mostly work fine, it still has some correctness issues [[1]](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=52f5d578d6c29bf254e93c69043b817d4047ca67), particularly on 9.6.

### Here's an alternate view of that same data that includes an idea how old the state is

```
SELECT a.datname,
         l.relation::regclass,
         l.transactionid,
         l.mode,
         l.GRANTED,
         a.usename,
         a.query,
         a.query_start,
         age(now(), a.query_start) AS "age",
         a.pid
FROM pg_stat_activity a
JOIN pg_locks l ON l.pid = a.pid
ORDER BY a.query_start;
```

For PostgreSQL older than 9.0:

```
  select a.datname,
         c.relname,
         l.transactionid,
         l.mode,
         l.granted,
         a.usename,
         a.current_query, 
         a.query_start,
         age(now(), a.query_start) as "age", 
         a.procpid 
    from  pg_stat_activity a
     join pg_locks         l on l.pid = a.procpid
     join pg_class         c on c.oid = l.relation
    order by a.query_start;
```

# Logging for later analysis

- If you suspect intermittent locks are causing problems only sometimes, but are having trouble catching them in one of these live views, setting the [log_lock_waits](http://www.postgresql.org/docs/current/static/runtime-config-logging.html#GUC-LOG-LOCK-WAITS) and related [deadlock_timeout](http://www.postgresql.org/docs/current/static/runtime-config-locks.html#GUC-DEADLOCK-TIMEOUT) parameters can be helpful. Then slow lock acquisition will appear in the database logs for later analysis.

# See also
