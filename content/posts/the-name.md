---
title: "Why u Stands for Micro: How the ukue Job Queue Got Its Name"
date: 2026-10-06
draft: false
featured_image: "https://ukue.com/images/micro-u.png"
tags: ["the name", "ukue"]
---

ukue is meant to be read as µkue, a micro queue. Plenty of people will say "you-queue" instead, and that's fine. The u is a stand-in for the real sign of micro, the Greek letter μ, and the story of how one turned into the other is older than computers.

Micro comes from the Greek *mikrós*, "small", and in Greek that word starts with μ, their letter m. When the metric system gave its prefixes one-letter symbols, the Latin m was already taken by milli, a thousandth. So micro, a millionth, got the Greek m. That's why a microsecond is written µs and a microfarad µF.

![The Greek letter mu turning into the Latin letter u: 100 µF written as 100 uF on a capacitor, 40 µs as 40 us in a log, µTorrent's web address utorrent.com, and µkue at ukue.com.](https://ukue.com/images/micro-u.png)

## The Lookalike

A lowercase μ looks like a u with a tail hanging down on the left. Most typewriters had no μ key, and neither did the plain-text character sets early computers used. So engineers typed the closest letter, and u became the everyday spelling of micro wherever μ was hard to get at.

You still see it everywhere. Capacitors are printed "100uF". Logs and benchmarks print "us" for microseconds, and Go's duration parser accepts "us" alongside "µs". MicroPython named its slimmed-down standard modules ujson and utime. The browser extension uBlock Origin started out as µBlock.

## µTorrent

The best-known example is µTorrent, the BitTorrent client released in 2005. Its μ really did mean micro: it was written to use very little memory at a time when other clients were heavy. But a web address can't hold a μ, so the site was utorrent.com, and many people said "you-torrent".

## Kue

The second half of the name comes from Kue, a job queue for Node.js first released in 2011. It was well known in its day, with delayed jobs and retries with backoff. It kept its jobs in Redis, and it's no longer maintained.

So µkue means what it says: a small version of a queue, with the same basic features, and without a Redis server to run. The µ appears where it can, and ukue everywhere people have to type it, on the command line, in the Go package and in [the address of this site](https://ukue.com/).
