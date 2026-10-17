# WordPress and Webshell Probing (Rule 100100 / 31101)

## Summary

External hosts repeatedly requested WordPress paths and known or random-looking PHP filenames against my web server. They were most likely checking for webshells left behind on previously compromised sites.

## Observed Behavior

Requests included:

- /wp-content/plugins/hellopress/wp_filemanager.php
- /this_is_a_new_hello_world.php
- /wp-filemannger4h.php
- /wp-includes/ID3/about.php
- Random-named scripts such as /2EYLzmsyjO8.php, /dpack.php, /up.php, /ws.php, /doc.php
- /wp-admin/install.php?step=1

Notable sources:

| Source IP | Provider | Requests | Period (UTC) | Reputation (checked Oct 17) |
|---|---|---|---|---|
| 35.238.129.244 | Google Cloud (US) | 826 | Oct 14 to 16 | No reports found |
| 20.194.17.163 | Microsoft Azure (Korea) | 242 | Oct 14 to 16 | In 7 OTX pulses |
| 20.193.50.139 | Microsoft Azure (Australia) | 151 | Oct 15 | In 4 OTX pulses |
| 20.196.193.222 | Microsoft Azure | 145 | Oct 16 | Not checked |
| 52.141.7.184 | Microsoft Azure (Korea) | 115 | Oct 14 | In 1 OTX pulse |

The Azure hosts sent no user-agent at all. The Google Cloud host used a Chrome-on-Windows string.

## Outcome

- All requests returned HTTP 404
- I don't run PHP or WordPress on this server, so there was nothing to find

## MITRE ATT&CK Mapping

- Tactic: Reconnaissance
- Technique: Active Scanning, Vulnerability Scanning (T1595.002)
- Related: Persistence, Server Software Component: Web Shell (T1505.003). The scanners are looking for existing webshells.

## Analyst Notes

Three of the Azure hosts appear in AlienVault OTX threat-intel pulses. That is a weaker signal, since pulses are community-curated and cloud ranges are shared. None of the hosts I checked had DShield reports.

Probing for specific webshell filenames means the scanners are reusing lists collected from other victims. The requests are noisy but harmless against a static site. Wazuh flagged them correctly, and the only real work left is reducing the noise.
