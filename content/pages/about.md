---
title: "About"
type: page
draft: false
---

ukue.com is the home of ukue, read µkue, a micro queue: a job queue that lives in one file. It keeps background jobs, their retries, their start times and the jobs that failed for good in a single SQLite file. A program can embed it as a library, or run the one small `ukue` binary when other programs, in any language, need the same queue.

## Why It Exists

Most apps reach the point where some work has to happen later, or somewhere other than the web request: a welcome email, a resized photo, a webhook that must get through even if the other side is down for an hour. The usual answer is a queue server such as Redis or RabbitMQ. For a small team that's one more service to install, secure, watch, back up and pay for, sometimes only to send a few emails. ukue does the same job with nothing running beside the app.

## How It's Built

- **On SQLite.** The file is an SQLite database with a documented layout, and SQLite takes care of crash safety and locking. Every change to a job is one transaction, flushed to disk before ukue moves on.
- **The file format comes first.** The layout, and the steps for claiming and finishing jobs, are written up in [the file format](https://ukue.com/file-format/), so programs in other languages can share a file without going through ukue at all.
- **Library or binary.** In Go, ukue is a package you import. For everything else there's the `ukue` command, which runs a script for each job or serves a small [HTTP API](https://ukue.com/http-api/).
- **Tested by breaking it.** The tests kill worker processes at random moments while they work, then check that no job was lost and the file is intact. [The results](https://github.com/ukue-queue/ukue/tree/main/test/results) are published with the code.

## License

All the code is open source under the Apache License 2.0, on [GitHub](https://github.com/ukue-queue/ukue). You can use it, change it and build it into your own products.

## Questions People Ask

### Is it really free?

Yes. ukue is free to use, and the code is free to take under the Apache License 2.0.

### Which languages can use it?

Go programs embed it directly. Any language can use it through the HTTP API, and any language that can write SQLite can add jobs straight into the file. The repository has examples in Go and in plain Python, with nothing to install for the Python ones.

### Can workers run on other machines?

Yes, through `ukue serve`. The file itself should stay on one machine's local disk, because SQLite's locking doesn't work over network file systems.

### What happens when a worker crashes?

The job it held goes back on the queue when its lease runs out, after a minute by default, and runs again. That's called at-least-once delivery. A job can run twice if its worker dies after doing the work and before marking it done, so work that must never repeat, such as a payment, should carry its own check.

### How fast is it?

Fast enough for most small teams. On a two-core cloud machine, with every change flushed to disk, ukue added about 4,000 jobs a second, and a worker ran about 900 jobs a second. The disk sets the pace more than anything else.

### How do you say it?

"Micro queue" is the idea behind the name, and "you-queue" is how most people will read it. Both are fine. [The post about the name](https://ukue.com/why-u-stands-for-micro-how-the-ukue-job-queue-got-its-name/) explains where the u comes from.

### Is it connected with Kue or Kueue?

No. ukue is independent. Kue was a job queue for Node.js, first released in 2011 and no longer maintained, and ukue borrows its name and its basic ideas. Kueue is a Kubernetes project that decides when batch jobs get a cluster's resources, which is a different job altogether.

### Can I suggest something?

Please do. The [contact page](https://ukue.com/contact/) has the address.
