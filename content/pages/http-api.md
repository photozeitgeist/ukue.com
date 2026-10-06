---
title: "The ukue HTTP API"
url: "/http-api/"
type: page
draft: false
---

`ukue serve` puts a ukue file behind a small JSON API, so programs in any language, and on other machines, can add and run jobs. Go programs can mount the same API in their own HTTP server with the `server` package.

```sh
ukue serve jobs.ukue                              # listens on 127.0.0.1:7660
UKUE_TOKEN=s3cret ukue serve --addr :7660 jobs.ukue
```

## The Basics

- **JSON in and out.** Errors come back as `{"error": "..."}` with a 4xx or 5xx status.
- **Token.** When the server has a token, from `UKUE_TOKEN` or `--token-file`, every request except `/v1/health` must send `Authorization: Bearer <token>`. The server won't listen beyond your own machine without a token, unless you start it with `--allow-no-token`.
- **Durations** can be a number of seconds, such as `30` or `0.5`, or a string such as `"90s"` or `"1h30m"`.
- **Times** in answers are in UTC, such as `"2026-10-06T09:00:00.000Z"`.
- **Payloads.** A `payload` that is a JSON string is stored as its text, and any other JSON value is stored as compact JSON. Send binary data as `payload_base64`. Answers give the payload back as text when it is valid UTF-8, and as `payload_base64` otherwise.
- **Size.** A request may be up to 1 MiB, unless the server was started with a different `--max-body`.

## Add a Job

**POST /v1/jobs**

```json
{"queue": "email", "payload": {"to": "dana@example.com"}, "delay": "10m", "max_attempts": 5, "priority": 0}
```

Only `queue` is required. Give `delay` or `run_at`, an RFC 3339 time, but not both. The answer is `201` with `{"id": 42}`.

## Run Jobs

**POST /v1/claim**

```json
{"queue": "email", "lease": 60, "wait": 20}
```

This takes the next due job and holds it for `lease` seconds, 60 by default. With `wait`, the request waits up to that long, at most 30 seconds, for a job to arrive. The answer is `200` with the job, or `204` with an empty body when there's nothing to do.

```json
{
  "id": 42,
  "queue": "email",
  "payload": "{\"to\":\"dana@example.com\"}",
  "attempt": 1,
  "max_attempts": 5,
  "priority": 0,
  "token": "6f1c0e2b9d4a47c3a1e5b8f2d7c90a13",
  "lease_until": "2026-10-06T09:01:00.000Z",
  "created_at": "2026-10-06T09:00:00.000Z",
  "last_error": ""
}
```

Keep the `token`. The four calls below need it, and it proves the worker still holds the job.

| Request | Body | Does |
|---|---|---|
| `POST /v1/jobs/{id}/ack` | `{"token": "..."}` | Marks the job done |
| `POST /v1/jobs/{id}/fail` | `{"token": "...", "error": "SMTP said 451", "retry_in": 30}` | Fails the attempt. The job is tried again after the usual backoff, or after `retry_in`. Send `"dead": true` to move it straight to the dead letters |
| `POST /v1/jobs/{id}/extend` | `{"token": "...", "lease": 60}` | Renews the lease for a long job |
| `POST /v1/jobs/{id}/release` | `{"token": "..."}` | Puts the job back without counting the attempt, for a worker that is shutting down |

If the worker no longer holds the job, because its lease ran out or someone deleted it, these calls answer `409`. The job may already be running somewhere else.

## Look and Tidy Up

<div style="overflow-x:auto">

| Request | Does |
|---|---|
| `GET /v1/stats` | Counts per queue: ready, delayed, running, expired, dead and done, with the oldest due time |
| `GET /v1/jobs?queue=&state=&after=&limit=` | Lists jobs in ID order. `state` is ready, delayed, running, dead or done. For the next page, pass the last ID as `after` |
| `GET /v1/jobs/{id}` | One job, in any state |
| `DELETE /v1/jobs/{id}` | Deletes a job |
| `POST /v1/jobs/{id}/retry` | Moves a dead job back to ready, with its attempts reset |
| `POST /v1/retry` | `{"queue": "email"}` retries every dead job in a queue, or in all queues without `queue` |
| `POST /v1/purge` | `{"state": "dead"}` or `{"state": "done"}` deletes those jobs, in one queue with `queue` |
| `GET /v1/health` | `{"ok": true, "version": "0.1.0", "format_version": 1}`, with no token needed |

</div>

## A Worker in a Few Lines

With curl and jq:

```sh
job=$(curl -s -X POST localhost:7660/v1/claim -d '{"queue": "email", "wait": 20}')
id=$(echo "$job" | jq -r .id)
token=$(echo "$job" | jq -r .token)
# ... do the work ...
curl -s -X POST "localhost:7660/v1/jobs/$id/ack" -d "{\"token\": \"$token\"}"
```

A complete worker in Python's standard library is [on GitHub](https://github.com/ukue-queue/ukue/blob/main/examples/python/http_worker.py), and the same reference is in [API.md](https://github.com/ukue-queue/ukue/blob/main/API.md).
