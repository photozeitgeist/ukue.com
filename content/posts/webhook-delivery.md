---
title: "Webhook Delivery With ukue: A Day of Retries, Then Dead Letters Instead of Lost Events"
date: 2026-10-06
draft: false
featured_image: "https://ukue.com/images/usecase-webhooks.png"
tags: ["use cases", "webhooks", "go"]
---

Say a small company runs an invoicing service written in Go. Its customers want to hear about payments the moment they happen, so the service sends webhooks: an HTTP POST with the event to a URL each customer chooses. Those URLs point at servers the company doesn't control. They go down for maintenance, get redeployed, hit rate limits and return errors. If the service sends each webhook once, straight from the code that records the payment, every one of those hiccups loses an event, and the customer finds out weeks later when their books don't add up.

Reliable delivery needs retries spread over hours, and somewhere to keep the events that never got through so a person can deal with them. A job queue gives both. Since the service is written in Go, ukue can run inside it and keep its jobs in one file on the same server.

![Thirty delivery attempts on a time line: after the first, they come 10 seconds, 30 seconds, 70 seconds and so on up to 85 minutes in, then once an hour until about 21 hours, after which an undelivered event becomes a dead letter.](https://ukue.com/images/usecase-webhooks.png)

## One Job per Delivery

When a payment arrives, the service adds a job for each customer endpoint that wants to hear about it:

```go
_, err := q.Enqueue(ctx, "webhooks", event, ukue.MaxAttempts(30))
```

ukue's default is 10 attempts over about an hour and a half, which is too short for an endpoint that's down overnight. After each failed attempt ukue waits 10 seconds, then 20, then 40, doubling up to a cap of an hour, plus up to a tenth at random so that thousands of retries don't all land in the same second. With 30 attempts, the waits add up to between 21 and 24 hours. A customer whose server was down all night still gets every event once it's back.

## The Worker Decides What Kind of Failure It Was

```go
err := q.Work(ctx, "webhooks", func(ctx context.Context, job *ukue.Job) error {
	var ev Event
	if err := json.Unmarshal(job.Payload, &ev); err != nil {
		return ukue.Permanent(err) // a broken payload won't fix itself
	}
	resp, err := deliver(ctx, ev) // POST with a 10-second timeout
	if err != nil {
		return err // timeout or refused connection: try again later
	}
	resp.Body.Close()
	switch {
	case resp.StatusCode < 300:
		return nil
	case resp.StatusCode == http.StatusGone:
		return ukue.Permanent(fmt.Errorf("%s answered 410 Gone", ev.URL))
	case resp.StatusCode == http.StatusTooManyRequests:
		return ukue.RetryAfter(fmt.Errorf("%s answered 429", ev.URL), retryAfter(resp))
	default:
		return fmt.Errorf("%s answered %d", ev.URL, resp.StatusCode)
	}
}, ukue.Concurrency(16))
```

Returning nil marks the job done. A plain error fails the attempt, and the job waits out its backoff before the next one. `ukue.Permanent` moves the job straight to the dead letters: a 410 means the customer took the endpoint down on purpose, and asking again tomorrow won't change that. `ukue.RetryAfter` sets the time of the next attempt directly, so when a customer's server answers 429 with a `Retry-After` header, the service waits as long as it was asked to. Both still count toward the 30 attempts.

`ukue.Concurrency(16)` runs 16 deliveries at once, so a few slow endpoints don't hold up everyone else's events. While a delivery runs, ukue keeps renewing its lease. If the process dies halfway through a request, the lease runs out within a minute, and the job goes back on the queue for the restarted process.

## Expect Some Duplicates

A job that runs at least once can run twice. If the process dies after a customer's server accepted an event but before ukue marked the job done, that event goes out again after the restart. This is normal for webhooks, and the fix is the usual one: every event carries a unique ID, so a receiver can ignore one it has already handled.

## Dead Letters Are a Support Tool

Events that ran out of attempts, or were marked permanent, stay in the file as dead jobs, each with its payload and last error. When a customer writes in to say their server was down for two days and asks for the missed events, support doesn't have to dig through logs. This command prints the dead deliveries with their payloads:

```sh
ukue list jobs.ukue --queue webhooks --state dead --json
```

A short script can pick out that customer's events from the list and send them back in line by ID:

```sh
ukue retry jobs.ukue 88121 88122 88140
```

A retried job starts again with its attempts reset. Inside the service, the same calls are `q.List` and `q.Retry`, which could sit behind a "resend failed events" button on the customer's dashboard.

## Where It Fits

A service sending thousands of webhooks a day is a light load for ukue. In its tests, one Go worker ran about 900 short jobs a second on a two-core cloud machine, and a webhook spends nearly all its time waiting on the network anyway. The file lives on one server, so if the service later runs on several machines, they can share the queue through `ukue serve` and its [HTTP API](https://ukue.com/http-api/).
