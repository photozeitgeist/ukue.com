---
title: "AI Transcription on a Rented GPU: Python Workers Pull Jobs From ukue Over HTTP"
date: 2026-10-06
draft: false
featured_image: "https://ukue.com/images/usecase-gpu-transcription.png"
tags: ["use cases", "http api", "python", "ai"]
---

Say a small podcast-hosting service wants to offer transcripts. Speech-to-text models are quick on a GPU and painfully slow without one, and the service's web server is a modest cloud machine with no GPU. So it rents a GPU server by the month from another provider, and needs a way to hand it work: here's an episode, transcribe it, report back when it's done. Episodes are long, so one job can take ten minutes. And the GPU server might be restarted, or swapped for a bigger one, at any time.

This is the case ukue's HTTP API is for. The queue file stays on the web server, `ukue serve` makes it reachable over the network, and the GPU machine runs a Python worker that asks for jobs over HTTP.

![A web server runs the web app, ukue serve and jobs.ukue. A GPU machine runs a Python worker with the speech model. The worker claims a job over HTTP, extends its lease every two minutes and acks it with the token, making only outgoing requests.](https://ukue.com/images/usecase-gpu-transcription.png)

## On the Web Server

```sh
ukue serve --addr 10.0.0.5:7660 --token-file /etc/ukue/token /srv/podcasts/jobs.ukue
```

`ukue serve` won't listen beyond its own machine without a token, unless it's told to with `--allow-no-token`, and every request must then carry the token as `Authorization: Bearer ...`. The server speaks plain HTTP, so here the two machines talk over a private network, where 10.0.0.5 is the web server's address. Across the open internet, a reverse proxy that adds HTTPS would sit in front.

When a host uploads an episode, the web app adds a job:

```sh
curl -X POST http://10.0.0.5:7660/v1/jobs -H "Authorization: Bearer $TOKEN" \
  -d '{"queue": "transcribe", "payload": {"episode": 31877}}'
```

## On the GPU Machine

The worker loads the speech model once, then loops over four HTTP calls:

1. **Claim.** `POST /v1/claim` with `{"queue": "transcribe", "lease": 300, "wait": 20}`. If a job is waiting, the answer comes at once, with the job and a token. If not, the request waits up to 20 seconds for one to arrive before it answers `204`. New episodes get picked up within moments, and an idle GPU doesn't hammer the server with requests.
2. **Extend.** A long episode can outlast the five-minute lease, so while the model runs, a background thread calls `POST /v1/jobs/{id}/extend` with the token every two minutes.
3. **Ack.** Once the transcript is saved where the web app can find it, `POST /v1/jobs/{id}/ack` with the token marks the job done.
4. **Fail.** `POST /v1/jobs/{id}/fail` with the token and an error. After a temporary problem, such as the audio failing to download, the job goes back in line, after the usual backoff or after `retry_in` seconds. For audio that's corrupt, `"dead": true` sends the job to the dead letters for a person to look at.

The repository has [a complete HTTP worker in plain Python](https://github.com/ukue-queue/ukue/blob/main/examples/python/http_worker.py) to start from, and the [HTTP API page](https://ukue.com/http-api/) lists every call.

The GPU machine only ever makes outgoing requests. It needs no open ports, no public address and no copy of the queue.

## When the GPU Machine Goes Away

The token is the worker's proof that it still holds the job. If the GPU machine crashes halfway through an episode, its extensions stop, the lease runs out, and the job goes back on the queue for the next claim, whether that comes from the same machine after a reboot or from its replacement. If a worker that lost its lease tries to ack the job anyway, the server answers `409`. The worker should then drop its result, because the job may already be running somewhere else. Saving transcripts under the episode number keeps a second run harmless: it just writes the same transcript again.

## More GPUs

A second GPU machine runs the same worker against the same server, and that's all it takes. Each claim takes SQLite's write lock before it picks a job, so two workers can't both pick the same one. The queue's part in each job is a handful of small HTTP requests, so one `ukue serve` on a modest web server can keep several GPUs busy.

The same pattern fits any slow job that needs hardware the web server doesn't have: video encoding, OCR on scanned documents, image models.
