# Wazuh Alert Review: October 2026

## Summary

I reviewed 8 days of Wazuh alert history covering my web server, SIEM host, Pi-hole host and LLM host. The goal was to separate real threats from noise after I set up a daily AI-generated digest.

## Numbers

- 11,533 alerts in the window
- 5,619 web alerts; 5,618 were HTTP 404 and one was 403
- No 2xx, 3xx or 5xx responses inside the web alerts, and none of Wazuh's "attack succeeded" rules fired
- Since I enabled real visitor IPs on Oct 13: 3,738 requests from 83 distinct public IPs
- 44 of the web requests were POSTs; all others were GET

## Who was scanning

Almost all heavy sources were cloud or VPS providers: Google Cloud, Microsoft Azure and DigitalOcean. Bursts of 100 to 270 requests in one to two minutes point to automated scanning.

Request types (all web alerts):

| Category | Count |
|---|---|
| Script / webshell probes | 1,903 |
| Secret / config / backup file probes | 1,745 |
| WordPress probes | 1,180 |
| Other | 654 |
| Admin / login panel probes | 128 |
| Traversal / injection attempts | 9 |

## Conclusion

This is background internet scanning. My site is a static page, so none of the probed paths exist, and no probe succeeded. This only covers error responses recorded in alerts, so it does not prove that no request ever returned 200.

## Changes I Made

- Apache now logs the real visitor IP through the Cloudflare header, trusted only from the local tunnel connector
- My daily digest groups routine alerts into counts and keeps anything unusual visible (POSTs, successful responses, unexplained file changes)
- I fixed a feedback loop where the digest's own output was logged and re-alerted
- Wazuh now ignores Pi-hole's database files
- I added Pi-hole as a DNS filter for ads, trackers, malware and phishing

## Limitations and Lessons

- IP reputation: I checked 12 scanner IPs against SANS ISC DShield and AlienVault OTX. One (213.209.159.133) is a known reported host (4 DShield reports, 30 OTX pulses). Three Azure addresses appear in a few OTX pulses (weaker evidence). The other eight, including the biggest Google Cloud sources, have no reports. Absence from these lists does not mean an IP is safe, so "malicious" here is mostly based on observed behavior
- The AI-written summary overstated some findings. For example, it called failed SSH logins from my own PC possible brute force, so I always check the data table underneath
- Real IPs in the logs make attribution possible, and I should have enabled them from the start

## Next Steps

- Add a Cloudflare rule blocking `.env`, `.git`, `wp-` and `.php` paths, since my site is static
- Evaluate Suricata on the web server's loopback to see decoded HTTP traffic
- Keep tuning the digest
