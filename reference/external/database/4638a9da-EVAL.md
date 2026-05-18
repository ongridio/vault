---
title: EVAL
source: https://redis.io/commands/eval
kind: external
domain: database
original_date: 2026-05-14
fetched_at: 2026-05-16
bookmark_title: EVAL – Redis
tags: [external, database]
---

> [!info] 外部文章 · 自动导入
> 来源：[redis.io](https://redis.io/commands/eval)
> 原始日期：2026-05-14
> 抓取日期：2026-05-16

# EVAL

#
EVAL

EVAL script numkeys [key [key ...]] [arg [arg ...]]

- Available since:
- Redis Open Source 2.6.0
- Time complexity:
- Depends on the script that is executed.
- ACL categories:
-
`@slow`

,`@scripting`

, - Compatibility:
- Redis Software and Redis Cloud compatibility

Invoke the execution of a server-side Lua script.

The first argument is the script's source code. Scripts are written in Lua and executed by the embedded Lua 5.1 interpreter in Redis.

The second argument is the number of input key name arguments, followed by all the keys accessed by the script.
These names of input keys are available to the script as the *KEYS* global runtime variable
Any additional input arguments **should not** represent names of keys.

**Important:**
to ensure the correct execution of scripts, both in standalone and clustered deployments, all names of keys that a script accesses must be explicitly provided as input key arguments.
The script **should only** access keys whose names are given as input arguments.
Scripts **should never** access keys with programmatically-generated names or based on the contents of data structures stored in the database.

**Note:**
in some cases, users will abuse Lua EVAL by embedding values in the script instead of providing them as argument, and thus generating a different script on each call to EVAL.
These are added to the Lua interpreter and cached to redis-server, consuming a large amount of memory over time.
Starting from Redis 7.4, scripts loaded with `EVAL`

or `EVAL_RO`

will be deleted from redis after a certain number (least recently used order).
The number of evicted scripts can be viewed through `INFO`

's `evicted_scripts`

.

Please refer to the Redis Programmability and Introduction to Eval Scripts for more information about Lua scripts.

## Examples

The following example will run a script that returns the first argument that it gets.

```
> EVAL "return ARGV[1]" 0 hello
"hello"
```


## Redis Software and Redis Cloud compatibility

| Redis Software |
Redis Cloud |
Notes |
|---|---|---|
| ✅ Standard |
✅ Standard |