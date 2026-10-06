---
title: "Quick Start"
url: "/quick-start/"
type: page
draft: false
---

ukue is a job queue that lives in one file. This page takes you from nothing to jobs running in a few minutes, from the command line, from Go, and from any other language.

## 1. Get the ukue Command

Download the archive for your system from [the download page](https://ukue.com/download/), unpack it and put `ukue` somewhere on your PATH. Check it works:

```sh
ukue version
```

Go programmers who only want the library can skip this and go to step 4.

## 2. Add Some Jobs

A ukue file holds any number of queues. Create one file and add two jobs to a queue called `email`, the second one for an hour from now:

```sh
ukue init jobs.ukue
ukue add jobs.ukue email '{"to": "dana@example.com"}'
ukue add jobs.ukue email --delay 1h '{"to": "sam@example.com"}'
ukue stats jobs.ukue
```

`ukue stats` shows one job ready and one delayed. The payload can be anything: JSON, plain text or bytes. ukue never looks inside it.

## 3. Run Them

`ukue work` takes jobs from a queue and runs a command for each one, with the payload on the command's standard input:

```sh
ukue work jobs.ukue email -- ./send-email.sh
```

The command tells ukue how it went through its exit status:

- **0** marks the job done.
- **65** says the payload is bad and trying again won't help. The job goes straight to the dead letters.
- **Anything else** fails the attempt. The job goes back on the queue and is tried again after 10 seconds, then 20, then 40, doubling up to an hour, 10 attempts in all.

The command also gets `UKUE_JOB_ID`, `UKUE_QUEUE`, `UKUE_ATTEMPT` and `UKUE_MAX_ATTEMPTS` in its environment. Add `--concurrency 4` to run four jobs at once, and `--timeout 2m` to stop a command that hangs. Press Ctrl-C to stop the worker: it lets running commands finish first.

To see what failed and try it again:

```sh
ukue list jobs.ukue --state dead
ukue retry jobs.ukue --all
```

## 4. Use It From Go

```sh
go get github.com/ukue-queue/ukue
```

```go
import (
	"github.com/ukue-queue/ukue"
	_ "github.com/mattn/go-sqlite3"
)

q, err := ukue.Open("jobs.ukue")
if err != nil {
	log.Fatal(err)
}
defer q.Close()

q.Enqueue(ctx, "email", []byte(`{"to":"dana@example.com"}`))

err = q.Work(ctx, "email", func(ctx context.Context, job *ukue.Job) error {
	return sendEmail(ctx, job.Payload)
}, ukue.Concurrency(4))
```

A handler that returns nil marks the job done, and an error fails the attempt. Wrap the error in `ukue.Permanent` to skip the retries, or in `ukue.RetryAfter` to pick the delay yourself. ukue works with any `database/sql` SQLite driver, and the examples use go-sqlite3.

## 5. Use It From Any Language

Start the server:

```sh
ukue serve jobs.ukue
```

It listens on `127.0.0.1:7660`. Add a job, then claim it as a worker would:

```sh
curl -X POST localhost:7660/v1/jobs  -d '{"queue": "email", "payload": {"to": "dana@example.com"}}'
curl -X POST localhost:7660/v1/claim -d '{"queue": "email", "wait": 20}'
```

The claimed job comes with a token. Send it back to `/v1/jobs/{id}/ack` when the work is done, or to `/v1/jobs/{id}/fail` when it isn't. The [HTTP API page](https://ukue.com/http-api/) covers every endpoint, and there's a [complete Python worker](https://github.com/ukue-queue/ukue/blob/main/examples/python/http_worker.py) that needs nothing but the standard library.

To let machines other than your own reach the server, set a token with `UKUE_TOKEN` or `--token-file`, and listen on another address with `--addr`.

## Where to Go Next

- [The file format](https://ukue.com/file-format/), for adding jobs straight into the file from another language.
- [The README on GitHub](https://github.com/ukue-queue/ukue#readme), the full manual with every command and option.
- [What a job queue is](https://ukue.com/what-is-a-job-queue-background-jobs-retries-and-dead-letters-explained/), if you're new to the idea.
