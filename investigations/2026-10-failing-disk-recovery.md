# Failing Host Drive: Detection, Backup and Live Migration (October 2026)

## Summary

While testing a new language model, a model file failed to load with a read error. I traced it to the only drive in my Proxmox host, a 500 GB laptop hard drive, which turned out to be failing. I backed up the important VMs, verified the backups, and moved all four VMs to a healthy drive while they kept running. No data was lost and no VM went down during the moves.

## Detection

The model server on my LLM VM stopped loading a freshly downloaded file:

```
llama_model_load: error loading model: read error: Input/output error
```

The VM's kernel log showed repeated read failures at the same sector, and the Proxmox host showed the matching errors on its physical drive:

```
I/O error, dev sda, sector 398485408 op 0x0:(READ)
```

The same sector failing again and again pointed to the disk itself, not the file or the download.

## Evidence (SMART)

| Attribute | Value | Meaning |
|---|---|---|
| Overall self-assessment | PASSED | Only fails once a vendor threshold is crossed, so it is not reassuring here |
| Reallocated sectors | 239 | Sectors already swapped for spares (healthy drives show 0) |
| Reported uncorrectable | 645 | Reads that failed and could not be recovered |
| Current pending sectors | 19 | Sectors that cannot be read right now |
| Offline uncorrectable | 19 | Confirmed unreadable sectors |

The "PASSED" status alongside these numbers was the main lesson: the raw counters matter more than the summary.

## Response

1. **Stopped the load test** so the bad sector was not read over and over, and brought the normal model server back up.
2. **Tested two USB drives** I had available (SMART short self-tests, both passed with zero bad sectors).
3. **Backed up the web, SIEM and Pi-hole VMs** with Proxmox's backup tool (`vzdump`, snapshot mode, so the VMs stayed up). No read errors occurred during the backups.
4. **Copied the backups to a second drive** and checked each archive with `zstd -t`, so there were always two verified copies before I erased anything.
5. **Created an LVM-thin storage pool** on the faster drive (USB 3) and moved each VM's disk with Proxmox's live **Move Disk**, one VM at a time, checking each service afterwards (DNS answers and blocking, website returns 200, Wazuh services and agents up, model server healthy).
6. **The LLM VM's move failed** at the bad sector, as expected, and Proxmox rolled it back safely. I deleted the damaged model file inside the VM, enabled `discard` on the virtual disk, ran `fstrim` so the freed blocks were released on the host (thin pool usage dropped by about 17 GB), and the retry no longer touched the bad area and completed.
7. **Scheduled weekly backups** of the web, SIEM and Pi-hole VMs to the second drive, keeping the last 3 plus 4 weekly copies.

## Outcome

- All four VMs now run from a healthy drive; the failing drive only holds the Proxmox OS and the old VM copies (kept as a fallback for a few days)
- Verified backups exist on a separate drive, with a weekly schedule
- No data loss and no VM downtime during the migration (one planned restart of the LLM VM to apply the disk setting)

## Lessons

- The overall SMART status said PASSED on a drive with hundreds of bad sectors. I now look at reallocated, pending and uncorrectable counts directly.
- I had no backups before this. Backups now run weekly and I verify archives, not just their existence.
- USB speed matters: the drive first connected over a USB 2.0 cable (480 Mb/s). Swapping to a USB 3 cable brought it to 5 Gb/s before I moved any VMs onto it.
- Thin provisioning plus TRIM can route a migration around unreadable blocks, if the damaged data is something that can be deleted.

## Next Steps

- Replace the internal drive with an SSD and move the Proxmox OS off the failing disk
- Add SMART monitoring that alerts me (reallocated or pending sector count increasing)
- Practice a full restore from backup so I know the recovery time
