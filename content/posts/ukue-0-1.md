---
title: "A Job Queue in One SQLite File: ukue 0.1 Lets Small Teams Skip Redis"
date: 2026-10-06
draft: false
featured_image: "https://ukue.com/images/ukue-terminal.png"
tags: ["releases", "job queues", "sqlite"]
---

Sooner or later every web app needs to do something outside the request. Send the welcome email after the user has seen the page. Resize the photo while they carry on. Retry the webhook when the other side is down. The standard way to do that is a job queue, and the standard way to get a job queue is to run Redis or RabbitMQ next to your app.

ukue 0.1 puts the whole queue in one file instead. It's a SQLite database with a documented layout, and it holds every queue, every waiting job, every retry and every job that failed for good. A Go program imports ukue as a library. Everything else uses the one small `ukue` binary, which runs a script for each job or serves a small HTTP API.

![A terminal running ukue: two jobs added to the email queue, ukue stats counting one ready and one delayed, and ukue work sending the first to a script and marking it done.](https://ukue.com/images/ukue-terminal.png)

## What You Get

The features are the ones a job queue needs, and no more:

- **Retries with backoff.** A failed job waits 10 seconds, then 20, then 40, doubling up to an hour, and gets 10 attempts by default.
- **Delayed jobs.** Add a job to run in an hour, or at a set time.
- **Dead letters.** A job that keeps failing, or fails in a way retrying won't fix, is set aside for a person to look at and retry.
- **Leases.** A worker holds each job it takes for a limited time and renews it while it works. If the worker crashes, the lease runs out and the job goes back on the queue.
- **Priorities**, so urgent jobs jump the line.

Every change to a job is one SQLite transaction, flushed to disk before ukue moves on. A crash at any moment leaves the queue consistent, and a job that was added survives a power cut.

## Why One File

A queue server is one more thing to install, secure, monitor, back up and pay for. For a team running one app on one or two machines, that's a lot of machinery to send emails in the background. Ruby on Rails reached the same conclusion: since Rails 8, new apps keep their jobs in the database by default. Most other languages still reach for Redis, or for a small queue library tied to that one language.

ukue takes the SQLite route and makes it language-neutral. The file's layout and the rules for claiming and finishing jobs are written up in [the file format](https://ukue.com/file-format/), so a Python script can add jobs to the same file a Go worker drains. And because it's SQLite, any SQLite tool can open the file to look inside, and copying it is a backup.

## Three Ways to Use It

From the command line, `ukue work` turns any script into a worker:

```sh
ukue add jobs.ukue email '{"to": "dana@example.com"}'
ukue work jobs.ukue email -- ./send-email.sh
```

The script gets the payload on its standard input. Exit status 0 means done, 65 means the payload is bad and the job should go to the dead letters, and anything else means try again later.

From Go, it's a package with `Enqueue` and `Work`. From any other language, `ukue serve` offers [an HTTP API](https://ukue.com/http-api/) where workers claim a job, do it and report back. The [quick start](https://ukue.com/quick-start/) covers all three.

## Where It Fits

ukue is built for small teams and single machines: one app server, or a few workers on the same box, handling up to several hundred jobs a second. Workers on other machines can share the queue through `ukue serve`. It isn't trying to be a distributed message broker. A system that has to move tens of thousands of messages a second across a cluster wants a different tool.

The code is open source under the Apache License 2.0, and the binaries for Linux, macOS and Windows are on [the download page](https://ukue.com/download/). [How it was tested](https://ukue.com/how-the-ukue-job-queue-was-tested-250-killed-workers-0-lost-jobs/) is a post of its own.
