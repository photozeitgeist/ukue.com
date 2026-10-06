---
title: "Appointment Reminders With ukue: Jobs Scheduled Days Ahead, Deleted When a Booking Moves"
date: 2026-10-06
draft: false
featured_image: "https://ukue.com/images/usecase-appointment-reminders.png"
tags: ["use cases", "delayed jobs", "http api"]
---

Picture a hair salon in London that takes bookings online, weeks ahead, and texts every client a reminder 24 hours before the appointment. No-shows cost it money, so the reminders matter. The obvious way to send them is a cron job that wakes up every few minutes and searches the bookings for appointments about a day away. That works until two runs overlap and a client gets two texts, or the server is down at the wrong moment and an hour of reminders never goes out, or a booking moves and the reminder still names the old time.

A delayed job is a cleaner fit. When the booking is made, the booking system adds a job that can't run until 24 hours before the appointment. The job waits in the queue, and a worker sends the text when its time comes.

![A timeline. On November 1 booking 4182 is made and reminder job 977 is set for 24 hours before the appointment. On November 6 the booking moves, job 977 is deleted and job 1203 is added. On November 20 at 2 p.m. job 1203 runs and the text goes out.](https://ukue.com/images/usecase-appointment-reminders.png)

## Adding the Reminder

The booking system is written in PHP and talks to ukue over its [HTTP API](https://ukue.com/http-api/), served on the same machine:

```sh
ukue serve /srv/salon/jobs.ukue     # listens on 127.0.0.1:7660
```

When a client books a cut for 2 p.m. on November 14, the system adds a job with a start time. As a curl command, the request looks like this:

```sh
curl -X POST localhost:7660/v1/jobs \
  -d '{"queue": "reminders", "payload": {"booking": 4182}, "run_at": "2026-11-13T14:00:00Z"}'
```

London is on UTC in November, so the times match. The answer is `201` with the job's ID, `{"id": 977}`, and the system stores 977 on the booking. The payload holds the booking number and nothing else: no time, no phone number. The reason comes up in a moment.

## When the Booking Moves

Clients reschedule all the time. When booking 4182 moves to the next week, the system deletes the old job and adds a new one:

```sh
curl -X DELETE localhost:7660/v1/jobs/977
curl -X POST localhost:7660/v1/jobs \
  -d '{"queue": "reminders", "payload": {"booking": 4182}, "run_at": "2026-11-20T14:00:00Z"}'
```

A cancelled booking just deletes its job.

## Sending the Text

The worker is a short PHP script, run by `ukue work`:

```sh
ukue work /srv/salon/jobs.ukue reminders -- php send-reminder.php
```

It reads the booking number from standard input and loads the booking fresh from the database before it does anything else. That's why the payload carries only the number: the script always works from the booking as it stands now. A cancelled booking, or one that has moved more than a day away, means this reminder is out of date, and the script exits with status 0, which marks the job done without sending anything. So even if the app crashed halfway through a reschedule and left the old job behind, no client gets a text with the wrong time.

If the text-message provider is down, the script exits with status 1, and ukue tries again after 10 seconds, then 20, then 40, doubling up to an hour. A reminder that's 20 minutes late is still useful. One that would arrive after the appointment isn't, so once the appointment has started, the script gives up with status 65. That moves the job to the dead letters, where the front desk can see which clients never got their reminder.

## A Long Schedule Costs Nothing

A busy salon might have thousands of reminders waiting weeks ahead. ukue keeps waiting jobs in an index sorted by start time, so finding the next one that's due doesn't slow down as the schedule grows. In ukue's tests, with 100,000 jobs scheduled for later, a worker's check for due work took 21 microseconds.

And if the server is off at the moment a reminder falls due, nothing is skipped. The job is still in the file when the server comes back, and the worker picks it up straight away. The script's own check then decides whether it's still worth sending.

## What the Front Desk Sees

```sh
ukue list /srv/salon/jobs.ukue --queue reminders --state delayed
ukue list /srv/salon/jobs.ukue --queue reminders --state dead
```

The first command lists the reminders still waiting for their time, and the second the ones that couldn't be sent. That's the whole system: one file, `ukue serve` for the booking app, `ukue work` for the texts, and no cron job.
