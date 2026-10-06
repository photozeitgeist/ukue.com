---
title: "How the ukue Job Queue Was Tested: 250 Killed Workers, 0 Lost Jobs"
date: 2026-10-06
draft: false
featured_image: "https://ukue.com/images/kill-test.png"
tags: ["testing", "job queues"]
---

A job queue makes one promise above all others: a job that was added runs, even if the machine has a bad day. So the most important tests for ukue don't check the happy path. They kill workers in the middle of their work and look at what's left.

The tests run on every change to the code, on Linux and macOS, and the recorded output is published with the code in [test/results](https://github.com/ukue-queue/ukue/tree/main/test/results). The numbers below come from the recorded run, on a two-core cloud machine.

![The output of three of ukue's tests: 2,000 jobs shared by four worker processes each ran exactly once, 20 workers killed while holding a job all had their jobs finish on the second attempt, and 8,000 jobs survived 250 random SIGKILLs with none lost.](https://ukue.com/images/kill-test.png)

## Killing Workers at Random

The hardest test starts four worker processes on one file with 8,000 jobs, four jobs at a time each. Every few dozen milliseconds it picks a worker at random and kills it with SIGKILL, which no program can catch or clean up after, then starts a new one. A kill can land anywhere: in the middle of a job, while claiming one, or halfway through writing to the file.

After 250 kills and about 20 seconds, all 8,000 jobs were done. None was lost, none was left stuck, and SQLite's own integrity check found the file intact, both during the run and after it.

329 jobs ran more than once. That's expected: their worker was killed after it started the work and before it marked the job done, so the job's lease ran out and another worker picked it up. This is what at-least-once delivery means in practice, and it's why handlers should be safe to repeat.

## A Kill While Holding a Job

A second test is more deliberate. A worker claims a job and then hangs, and the test kills it. It checks that the job stays put while the lease is valid, then goes back on the queue once the lease runs out, and finishes on its second attempt. It does this 20 times in a row, and all 20 jobs came back.

## Many Processes, One File

Four processes with four workers each shared 2,000 jobs, with no kills. Every job ran exactly once. The claim step takes SQLite's write lock before it looks for a job, so two workers can't both pick the same one.

## Another Language at the Same Time

ukue's [file format](https://ukue.com/file-format/) is meant to be shared, so one test has Python add 200 jobs, through its own copy of SQLite, while Go workers take them from the same file. Every job ran once and the file stayed intact. The Python worker for the HTTP API was tested against the server as well.

## A Long Schedule

A queue full of jobs scheduled for later shouldn't slow down the jobs that are due. With 100,000 scheduled jobs in the file, a claim that found nothing due took 21 microseconds, and adding, claiming and finishing a job took 1.1 milliseconds. Claims look jobs up through indexes, and the test checks SQLite's query plans to make sure they stay that way.

## And the Rest

Every operation has its own tests: delays, priorities, backoff, dead letters, leases that run out, the HTTP API and the command line. One test has another program hold the file's write lock just as a job finishes; the worker keeps its lease and keeps trying to mark the job done until it can, and the job runs once. The whole suite also runs under Go's race detector, which found nothing. You can run all of it yourself with `sh scripts/record-tests.sh` from [the repository](https://github.com/ukue-queue/ukue).
