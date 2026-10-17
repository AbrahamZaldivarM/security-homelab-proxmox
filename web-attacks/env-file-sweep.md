# Environment File Sweep (Rule 31101 / 31516)

## Summary

Multiple external hosts ran automated sweeps against my web server looking for exposed environment and configuration files, including `/.env.swp` (a Vim swap file that can leak a copy of `.env`).

## Observed Behavior

Each sweep sent roughly 165 to 270 requests within one to two minutes. Requests included:

- /.env, /.env.local, /.env.production, /.env.bak, /.env.old
- /.env.swp
- /api/.env, /backend/.env, /app/.env
- /.git/config
- /.docker/config.json
- /php_info.php

Source hosts (all hosted in cloud or VPS providers):

| Source IP | Provider | Date (UTC) | Reputation (checked Oct 17) |
|---|---|---|---|
| 188.166.255.144 | DigitalOcean (Singapore) | Oct 14 | No reports found |
| 188.166.191.103 | DigitalOcean (Singapore) | Oct 14 | No reports found |
| 35.202.166.200 | Google Cloud (US) | Oct 14 | No reports found |
| 213.209.159.133 | Feo Prest SRL (Germany) | Oct 15 | **Reported**: 4 DShield attack reports (Aug 30 to Oct 12), listed in 30 AlienVault OTX threat-intel pulses |
| 34.145.145.8 | Google Cloud (US) | Oct 16 | No reports found |

User-agents were generic desktop and mobile browser strings (Chrome on Windows, macOS, Linux and Android), most likely spoofed.

## Outcome

- Every request returned HTTP 404
- No environment, Git or Docker configuration was exposed
- My site is a single static page, so none of these files exist on the server

## MITRE ATT&CK Mapping

- Tactic: Reconnaissance
- Technique: Active Scanning, Vulnerability Scanning (T1595.002)
- Tactic: Credential Access (attempted)
- Technique: Unsecured Credentials, Credentials In Files (T1552.001)

## Analyst Notes

I checked the sources against SANS ISC DShield and AlienVault OTX. Only one of the five (213.209.159.133) is a known reported host. The others have no reports, which does not mean they are clean: these databases only know what other people have reported, and cloud addresses get reused.

This is opportunistic mass scanning for leaked secrets, not a targeted attack. The same file list is requested from many unrelated hosts, so it looks like the same tooling running from rented infrastructure. Keeping secrets out of the web root and blocking these paths at the edge (Cloudflare) removes the risk and cuts the log noise.
