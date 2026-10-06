---
title: "Checking 300 Websites Every Night With ukue, Cron and a Shell Script"
date: 2026-10-06
draft: false
featured_image: "https://ukue.com/images/usecase-website-checks.png"
tags: ["use cases", "command line", "monitoring"]
---

Say a two-person web agency looks after 300 client websites. Each morning they want to know which sites were down overnight and which TLS certificates are close to expiring, before a client calls about it. Monitoring services do this for a monthly fee per site. The agency already has a Linux server, a text file of domains and a shell script that checks one site, and would rather not add anything bigger than that.

The first version is a loop over the list, and it mostly works. But one slow site holds up the rest, a site that was down for 30 seconds during a restart gets reported as down, and if the loop dies at site 140, sites 141 to 300 go unchecked without anyone noticing. Putting each check in a job queue deals with every one of those, and with ukue it adds one file and one command to the server.

![Cron adds one job per site at 2 a.m., and ukue work runs eight checks at a time. Healthy sites finish, brief outages pass on a retry, and sites that stay down for an hour and a half or have expiring certificates end up in the dead letters, which ukue list shows in the morning.](https://ukue.com/images/usecase-website-checks.png)

## Every Night at Two

Cron runs a short script at 2 a.m.:

```sh
#!/bin/sh
# nightly.sh, run by cron: 0 2 * * * /srv/checks/nightly.sh
ukue purge --state dead --queue checks /srv/checks/jobs.ukue
while read -r site; do
  ukue add /srv/checks/jobs.ukue checks "$site"
done < /srv/checks/sites.txt
```

The first command clears out the previous night's results, which have been read by now. Then each domain becomes one job.

## The Check Itself

A worker runs all the time as a service, eight checks at once:

```sh
ukue work --concurrency 8 --timeout 1m /srv/checks/jobs.ukue checks -- /srv/checks/check-site.sh
```

```sh
#!/bin/sh
# check-site.sh: the payload is the site's domain
site=$(cat)

# Down or answering with an error? Fail, and ukue tries again later.
curl --fail --silent --show-error --max-time 20 --output /dev/null "https://$site/" || exit 1

# Certificate expiring within 14 days? Another try won't change that.
if ! echo | openssl s_client -connect "$site:443" -servername "$site" 2>/dev/null \
     | openssl x509 -noout -checkend 1209600 >/dev/null; then
  echo "certificate expires within 14 days" >&2
  exit 65
fi
```

A healthy site exits with status 0, and its job is done and deleted. A site that's down or answering with errors exits with 1, and ukue tries it again after 10 seconds, then 20, then 40, doubling up to an hour. A site that was in the middle of a restart passes on the second or third try, and nobody hears about it. Only a site that fails all 10 attempts, over about an hour and a half, ends up in the dead letters. An expiring certificate exits with 65, which sends the job to the dead letters at once, since checking again won't make the certificate any newer.

`--concurrency 8` means a slow site takes up one of eight slots instead of holding up the whole list, and `--timeout 1m` stops a check that hangs and counts it as a failed attempt.

## The Morning List

```sh
ukue list /srv/checks/jobs.ukue --queue checks --state dead
```

That's the to-do list for the day: every site that was down for over an hour and a half, and every certificate about to expire. `ukue show` on any of them prints the domain with the last error. For a script, that's its exit status plus the end of what it wrote to stderr, such as curl's "Could not resolve host" or "The requested URL returned error: 503". An empty list means a quiet night.

## When the Server Itself Has a Bad Night

If the agency's own server reboots at 3 a.m., the remaining checks are still in the file. `ukue work` starts again with the service and carries on where it stopped, and any check that was running when the server went down goes back on the queue once its lease runs out. During the night, `ukue stats` shows how many checks are left.

The whole system is a cron line, two short scripts, one file and one `ukue work` service. It grows with the list, too: 3,000 sites would be the same setup with a longer `sites.txt`.
