---
title: "The ukue File Format"
url: "/file-format/"
type: page
draft: false
---

A ukue file is an SQLite database with two tables in it. Any program that can write SQLite can add jobs to it, and a program that follows the steps on this page can claim and finish them too, alongside the `ukue` command and the Go package. This is version 1 of the format. The complete description, with every SQL statement, is [FORMAT.md on GitHub](https://github.com/ukue-queue/ukue/blob/main/FORMAT.md).

> **What stays the same.** The version is stored in the file. Within version 1, later releases may add indexes, settings and columns with default values, but nothing is removed or renamed, and no column changes meaning. A change that older programs couldn't handle safely gets a new version number, and ukue refuses to open a file whose version it doesn't know.

## The File

- **SQLite 3, in WAL mode.** ukue switches a new file to WAL, and the setting is stored in the file, so every program that opens it uses WAL too. Readers can then carry on while a writer commits.
- **Marked as ukue.** A file ukue creates for itself has the application ID `0x756B7565`, the letters "ukue", in its SQLite header. The tables can also sit inside an app's own database, next to its tables.
- **On one machine.** Many processes on one machine can use the file at once. Programs elsewhere go through `ukue serve`, because SQLite's locking doesn't work over network file systems.
- **Times** are whole milliseconds since 1970, in UTC.

## The Tables

`ukue_meta` holds settings, most importantly `format_version`, which is `1`. `ukue_jobs` holds one row per job:

<div style="overflow-x:auto">

| Column | What it holds |
|---|---|
| `id` | The job's number, never used twice |
| `queue` | The queue's name, 1 to 200 characters |
| `payload` | The job's data, as bytes. ukue never reads it |
| `state` | `ready`, `running`, `done` or `dead` |
| `priority` | Among due jobs, the highest priority runs first. Default 0 |
| `attempts` | How many times the job has been claimed |
| `max_attempts` | How many attempts it gets before it's dead. Default 10 |
| `run_at` | When a ready job may start. Default now |
| `lease_until` | When a running job's lease runs out |
| `token` | The random value that proves which worker holds a running job |
| `last_error` | What the last failed attempt reported |
| `created_at`, `updated_at` | When the job was added, and last changed |

</div>

Every column except `queue` and `payload` has a default, so adding a job can be this short:

```sql
INSERT INTO ukue_jobs (queue, payload) VALUES ('email', '{"to": "dana@example.com"}');
```

## The Steps

Each step is one transaction. Set `PRAGMA busy_timeout` so a writer waits for another one instead of failing, and `PRAGMA synchronous = FULL` so each commit is durable.

1. **Claim.** Start with `BEGIN IMMEDIATE`, which takes the write lock before reading, so two workers can never pick the same job. Put back any running job whose lease has run out. Then pick the due job with the highest priority, the oldest first, set it to `running` with one more attempt, a lease time and a new random token, and commit.
2. **Finish.** Delete the row, matching its ID and token. If no row matches, the worker no longer holds the job.
3. **Fail.** Set it back to `ready` with a later `run_at`, or to `dead` when it was the last attempt or trying again won't help, again matching the token.
4. **Renew.** A worker on a long job moves `lease_until` forward before it passes, matching the token.

Two rules hold it together. A running job is only ever changed together with its token. And nobody keeps a transaction open while doing a job's work: claim, commit, work, then finish or fail in a new transaction.

## Try It From Python

The repository has [a Python script](https://github.com/ukue-queue/ukue/blob/main/examples/python/add_job.py) that adds a job with nothing but Python's standard library:

```sh
python3 add_job.py jobs.ukue email '{"to": "dana@example.com"}'
```

The tests run it while Go workers take jobs from the same file at the same moment, and check that every job runs once and the file stays intact.
