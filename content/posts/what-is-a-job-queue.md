---
title: "What Is a Job Queue? Background Jobs, Retries and Dead Letters Explained"
date: 2026-10-06
draft: false
featured_image: "https://ukue.com/images/job-lifecycle.png"
tags: ["guides", "job queues"]
---

A job queue is a to-do list for software. When something has to happen but doesn't have to happen right now, a program writes it on the list and moves on. A separate worker takes items off the list and does them. That's the whole idea, and almost every web app ends up needing it.

Take a sign-up form. When a user signs up, the app has to save the account, send a welcome email, and maybe tell the sales team's CRM. Saving the account is quick. Talking to a mail server can take seconds, and it might be down. Making the user stare at a spinner while that happens is a poor trade, so the app saves the account, adds "send a welcome email to Dana" to a job queue, and shows the next page at once. A worker sends the email a moment later.

![How a job moves through ukue: added jobs wait as ready, a worker claims them as running, finished ones are done, failed ones go back to ready after a wait, and jobs whose last attempt failed become dead until a person retries them.](https://ukue.com/images/job-lifecycle.png)

## The Parts of a Job

A job is a small record. It names its **queue**, such as `email` or `thumbnails`, so different workers can handle different kinds of work. It carries a **payload**, the data the worker needs: an email address, a file name, an order number. And it keeps a little bookkeeping: when it may run, how many times it has been tried, and what went wrong last time.

## When Things Go Wrong

The list is the easy part. What makes a job queue worth having is how it copes with failure.

**Retries with backoff.** If the mail server is down, the job goes back on the list with a later start time. Each failure makes the wait longer, so a service that's struggling isn't hammered with retries. In [ukue](https://ukue.com/quick-start/) the waits are 10 seconds, then 20, then 40, doubling up to an hour.

**Dead letters.** Some jobs will never succeed. The address is malformed, or the account was deleted. After a set number of attempts, or straight away when the worker knows retrying is pointless, the job moves to a separate pile. Nothing is thrown away. A person can look at the pile, fix the cause and send the jobs back.

**Leases.** A worker can crash in the middle of a job, and the job must not vanish with it. So a worker holds each job under a lease, a time limit it renews while it works. If the worker dies, the lease runs out and the job goes back on the list for another worker.

## At Least Once

Leases raise a subtle point. Suppose a worker sends the email, then crashes before it can mark the job done. The lease runs out, and another worker sends the email again. A queue can promise that every job runs, or that no job runs twice, but not both when machines can crash at any moment. Most job queues, ukue included, promise the first, which is called at-least-once delivery.

That's usually fine: a second welcome email is a minor nuisance. Where a repeat would hurt, as with a payment, the job carries its own check, such as an ID the payment provider uses to ignore duplicates.

## Delayed and Scheduled Jobs

Not every job should run now. "Send a reminder in three days" or "close the trial on the 30th" are jobs with a start time. A good queue keeps them waiting without slowing down the jobs that are due. ukue keeps 100,000 scheduled jobs and still finds the next due one in microseconds, because it looks them up through an index instead of reading through the list.

## Where the List Lives

The list has to survive restarts and crashes, so it can't live in a program's memory. The usual choices are a queue server such as Redis or RabbitMQ, or the app's own database. ukue takes a third path: [one SQLite file](https://ukue.com/file-format/), embedded in the app or served by one small binary, with nothing else to run.
