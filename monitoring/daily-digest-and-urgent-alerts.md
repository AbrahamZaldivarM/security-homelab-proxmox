# Alerting Pipeline: Daily AI Digest and Instant Alerts

## Summary

Wazuh produces hundreds of alerts a day in my lab, almost all of them internet scanning against my website. I built two pieces on top of it: a **daily digest** that an LLM summarizes every morning, and an **instant alert** service that pings my phone on Discord for the few events that cannot wait. Both run on the Wazuh VM and only read Wazuh's alert files.

## Daily Digest (7:00 AM)

A systemd timer runs a Python script that:

1. **Reads the last 24 hours of alerts**, removing duplicates by alert ID (see "Bug I Fixed" below)
2. **Groups routine noise into counts** instead of listing each alert:
   - web scanning answered with 4xx responses
   - successful logins from my own trusted devices
   - sudo by my admin accounts
   - package updates, and file changes that match package activity on the same host within 3 hours
   - known false positives (for example rootcheck's "trojaned" `diff`)
3. **Keeps anything unusual visible**: failed logins, logins from sources not on my trusted list, web requests that did *not* get a 4xx, POST requests, and file changes with no matching update
4. **Computes "Verified facts" in code**: alert counts, web responses by HTTP status, trusted vs. untrusted logins, failed logins by source. These are printed above the AI's text, so I can check the summary against numbers the model did not produce.
5. **Asks the local LLM** (Qwen3.6-35B-A3B, thinking on) for an overview, what needs attention, patterns, uncertainty and suggested checks
6. **Posts to a private Discord channel** and saves a copy on the server

### Treating log data as untrusted

Log lines contain attacker-controlled text (URLs, user agents). Before anything reaches the model:

- The data is wrapped in clearly marked "untrusted log data" boundaries
- Any copy of those boundary markers, or fake "verified facts" headings, inside the logs is stripped out
- Control characters are removed and samples are length-limited

If the model is unreachable or returns nothing, the digest still posts the verified facts and raw data with a clear note.

### Bug I fixed: double counting

Wazuh's `alerts.json` is a **hard link** to the current day's archive file (`ossec-alerts-DD.json`). My first version read both, so the current day's alerts were counted twice. In one check, 1,508 records contained only 846 unique alert IDs. The digest now skips files it has already read (by device and inode) and removes duplicate alert IDs. I also re-checked my [October alert review](../investigations/2026-10-web-alert-review.md) for the same problem.

## Instant Alerts

A separate always-on service follows `alerts.json` and sends a Discord message that mentions me when:

| Trigger | Why |
|---|---|
| Wazuh rule level 12 or higher | High-severity detections |
| A successful login from a source not on my trusted device list | Possible stolen credentials or unknown access |
| `sudo` used by an account that is not one of my admin accounts | Unexpected privilege use |

Design choices:

- Decisions are made by code, not the LLM, so an alert cannot be talked out of firing
- Repeats of the same event within 30 minutes are grouped into a single "+N more" follow-up
- A local rate limit (10 messages per 10 minutes) and retries that respect Discord's rate limits
- Every sent or failed alert is logged on the server
- The service runs as an unprivileged user with a sandboxed systemd unit and can only read the alerts

Before enabling it I replayed the previous 24 hours of alerts in a dry-run mode: exactly one event would have alerted: a password login to my web server from an address not on my trusted list. I checked it, and it was me. That is still the right call for an alert: a login I cannot immediately place is exactly what I want on my phone.

## Analyst Notes

The LLM is useful for turning hundreds of alerts into a readable morning summary, but it has overstated findings before. That is why every number that matters is computed by code and shown next to the AI text, and why the urgent path does not use the AI at all.
