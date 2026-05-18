---
title: Slow Query Questions
source: https://wiki.postgresql.org/wiki/Slow_Query_Questions
kind: external
domain: database
author: PostgreSQL Project
license: postgresql-license
fetched_at: 2026-05-18
tags: [external, database]
---

> [!info] External article · imported reference
> Source: [wiki.postgresql.org](https://wiki.postgresql.org/wiki/Slow_Query_Questions)
> Author: PostgreSQL Project
> License: postgresql-license
> Fetched: 2026-05-18

# Slow Query Questions

## Guide to Asking Slow Query Questions

In any given week, some 50% of the questions on #postgresql IRC and 75% on pgsql-performance are requests for help with a slow query. However, it is rare for the requester to include complete information about their slow query, frustrating both them and those who try to help.

Please post performance related questions to the pgsql-performance mailing list or to the IRC channel.

Note: if asking for help on IRC, post relevant information onto a paste site like <https://paste.depesz.com/> and not directly on IRC.

See also: [Guide to reporting problems](/wiki/Guide_to_reporting_problems "Guide to reporting problems")

## Things to Try Before You Post

You will save yourself a lot of time if you try the following things before you post your question:

- Read [Using EXPLAIN](/wiki/Using_EXPLAIN "Using EXPLAIN") if you haven't already.
- ANALYZE relevant tables to ensure statistics are up-to-date;
  - Collects updated stats on table size (ntuples, npages) and column values (nullfrac, ndistinct, MCVs, and histogram).
  - Note that autovacuum does not currently handle analysis of a parent due to changes to or of its children. A parent table should be manually ANALYZEd after contents of its child tables changes significantly, such as after a DROP or ALTER..DETACH/ATTACH/UN/INHERIT, or bulk DELETE or UPDATE or many INSERTs.
- VACUUM relevant tables to avoid bloat and set relallvisible;
- REINDEX relevant indices - resolves dead index tuples, bloat, and new index will have entries in order of heap TID; note, this will block queries (unless you use REINDEX CONCURRENTLY in v12+; see also pg_repack)
- Check your main GUC settings to make sure that they are set to sensible values (see [Tuning Your PostgreSQL Server](/wiki/Tuning_Your_PostgreSQL_Server "Tuning Your PostgreSQL Server") for additional hints):
  - shared_buffers should be 10% to 25% of available RAM
  - effective_cache_size should be 75% of available RAM
- Test if you can reproduce the issue with a smaller query, or with different query parameters;
- Test changing work_mem: increase it to 8MB, 32MB, 256MB, 1GB. Does it make a difference?
- For Insert/Update/Delete queries, you should also try configuring your WAL:
  - Move pg_wal to a separate storage device, if possible
  - Increase max_wal_size to 2GB or more (assuming you have disk space)

## Information You Need To Include

Please plan to spend ~30 minutes collecting needed info and preparing your question. Thousands of people will see it, but if you don't make
it easy for people to help you, they may not respond.
If it's well-written and provides needed information, most requests receive helpful responses. Some even lead to bugfixes or other patches. But you're unlikely to receive much unpaid help from the community without a significant effort on your part to provide as much information as may be relevant.

### Postgres version

Please provide the exact server version you are using (`SELECT version();` is an easy way to get it). The behavior of the planner changes in every release, so this is important.

### Operating system+version

What OS / version ? At least for linux, you can get the distribution by running:
`tail /etc/*release`

### Full Table and Index Schema

Post the definitions of all tables and indexes referenced in the query. If the query touches views or custom functions, we'll need those definitions as well.
Run psql command "\d table" with the tables/views/indices referenced in the problem query.

### Table Metadata

In addition to the table definition, please also post the approximate number of rows in the table(s):

- SELECT relname, relpages, reltuples, relallvisible, relkind, relnatts, relhassubclass, reloptions, pg_table_size(oid) FROM pg_class WHERE relname='TABLE_NAME';

Does the table have anything unusual about it?

- contains large objects
- has a large proportion of NULLs in several columns
- receives a lot of UPDATEs or DELETEs regularly
- is growing rapidly
- has many indexes on it
- uses triggers that may be executing database functions, or is calling functions directly

### EXPLAIN (ANALYZE, BUFFERS), not just EXPLAIN

`EXPLAIN (ANALYZE, BUFFERS, SETTINGS)` tells us how the query actually was executed, not just how it was planned. This is essential information in finding out how the planner was wrong, if anywhere. Example: `EXPLAIN (ANALYZE, BUFFERS) select * from tablename;` If you can't run an EXPLAIN (ANALYZE, BUFFERS) because the query never completes, then say so.

- It may be best to attach the output as a separate file, if your mail client is likely to mess up the formatting. (Otherwise, people may be discouraged from helping you by the need to reformat the mail before it's readable).
- Alternately, paste your `EXPLAIN (ANALYZE, BUFFERS, SETTINGS)` result into [explain.depesz.com](http://explain.depesz.com/) and post links to the resulting pages. We love this, because it makes the plans much easier to read and examine.
- Turn on track_io_timing before executing the EXPLAIN. If you log in as a superuser, you can turn on track_io_timing for only the current session by executing **`SET track_io_timing = on;`**
- If using PostgreSQL v11 or earlier, SETTINGS is not supported and you should also provide a list of all your nondefault settings. See [Server Configuration](/wiki/Server_Configuration "Server Configuration").

### History

Was this query always slow, or has it gotten slower over time? If the plan/execution of the query used to be different, do you have copies of those query plans? Has anything changed in your database other than the accumulation of data?

### Hardware

Please post the essential information about your hardware platform. If any elements of your hardware are unusual, please include detailed information on those. See the [Guide to reporting problems](/wiki/Guide_to_reporting_problems "Guide to reporting problems") for what sort of hardware information is useful.

### Hardware benchmark

- Run bonnie++ or other drive speed tests to see if your performance problem isn't simply hardware-based. For example, a RAID-5 configuration will never have fast inserts/updates no matter how you play with your queries.
  - bonnie++ -f -n0 -x4 -d /var/lib/pgsql
- Test sequential read speed of the device where the postgres data files are stored, run this several times. If you have a separate drive for WAL, or tmp, or multiple tablespaces, run several times for each. 128*$RANDOM/32 is 128GB, which needs to be proportionally increased if you have more than ~64GB RAM. 32K is 32GB, which needs to be increased if your drives are very fast (or very slow). If your DB is very small, this may not be a good test (but in that case, drive speed maybe doesn't matter anyway).
  - time dd if=/dev/sdX2 of=/dev/null bs=1M count=32K skip=$((128*$RANDOM/32))

### Maintenance Setup

Are you running autovacuum? If so, with what settings? If not, are you doing manual VACUUM and/or ANALYZE? How often?
SELECT * FROM pg_stat_user_tables WHERE relname='table_name';

### WAL Configuration

For data writing queries: have you moved the WAL to a different disk? Changed the settings?

### GUC Settings

What database configuration settings have you changed? What are their values? (These are things like "shared_buffers", "work_mem", "enable_seq_scan", "effective_io_concurrency", "effective_cache_size", etc). See [Server Configuration](/wiki/Server_Configuration "Server Configuration") for a useful query that will show all of your non-default database settings, in an easier to read format than posting pieces of your postgresql.conf file.

### Statistics: n_distinct, MCV, histogram

Useful to check statistics leading to bad join plan.
SELECT (SELECT sum(x) FROM unnest(most_common_freqs) x) frac_MCV, tablename, attname, inherited, null_frac, n_distinct, array_length(most_common_vals,1) n_mcv, array_length(histogram_bounds,1) n_hist, correlation FROM pg_stats WHERE attname='...' AND tablename='...' ORDER BY 1 DESC;

### Enable Logging

Some suggested values to start with:

- log_min_duration_statement = '2222ms'
- log_autovacuum_min_duration = '9s'
- log_checkpoints = on
- log_lock_waits = on
- log_temp_files = 0

- log_destination = 'stderr,csvlog'
- log_rotation_age = '2min'
- log_rotation_size = '32MB'
- logging_collector = on
