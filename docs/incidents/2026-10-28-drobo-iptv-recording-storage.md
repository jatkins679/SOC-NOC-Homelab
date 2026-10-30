# IPTV recording share went down — October 28, 2026

**Storage:** `storage01` — `192.168.1.208`  
**Docker VM / Channels DVR:** `192.168.1.174`  
**Status:** Share working again; Drobo drive problem not fixed.

## The problem

My IPTV recording share stopped responding, and Uptime Kuma's **IPTVX Recording Storage** monitor (#31) went red.

On `docker`, `/mnt/iptv-recordings` was failing with I/O errors. Even basic commands such as `df` and a test-file write failed. The SMB server was still reachable on port 445, and the separate `CHANNELS` movie share on E: was fine. The problem appeared to be with the G: recording volume.

## What I checked

On `storage01`, Windows logged several disk and NTFS problems around 2:29–2:34 PM PDT:

- Event 50: delayed writes failed, including NTFS metadata.
- Event 140: transaction-log flush errors.
- Events 51 and 153: disk I/O errors and retries.
- Windows also reported the Drobo USB device as not present.

At first, Windows couldn't find G: or `G:\IPTV\Recordings`, even though the SMB share was still listed.

The Drobo eventually appeared again. Windows showed G: as NTFS Healthy, and `Test-Path 'G:\IPTV\Recordings'` returned True.

Drobo Dashboard showed a separate concern: **Bay 1's 6 TB Seagate drive had failed** (blinking red). Bay 0's 6 TB drive showed healthy.

Back on `docker`, the CIFS share became accessible again. `df` reported about 1.3 TB used, and the storage check passed. Uptime Kuma returned to green. I didn't have to restart Channels DVR or remount the share manually.

The reported **64 TB** size is the Drobo's virtual volume size. It should not be treated as available physical storage.

## Storage setup

| System | Path or share |
| --- | --- |
| `storage01` | `IPTV-Recordings` → `G:\IPTV\Recordings` |
| `docker` | `/mnt/iptv-recordings` (CIFS mount) |
| `storage01` | `CHANNELS` movie share on E: (unaffected during this outage) |
| `docker` | `/usr/local/bin/check-iptvx-storage` |
| `docker` | `iptvx-storage-monitor.timer` and Uptime Kuma `:3001` |

## Where it stands

The recording share is mounted and passed a basic read/write check, so the outage itself is over.

**That doesn't mean the Drobo is healthy.** Bay 1 still needs attention, and I haven't confirmed the integrity of every recording or the array's remaining protection. Windows reporting the volume as Healthy isn't the same as checking all the data.

I'm not buying a $200-plus replacement drive for this aging Drobo. I'd rather move to a proper NAS, so protecting and moving the recordings is the priority.

## Next time / follow-up

- Check the Drobo's bay status and Windows disk/NTFS events for new errors.
- Keep the storage monitor running and confirm it stays green.
- Back up the important recordings to healthy storage before attempting filesystem repairs or large migrations.
- Don't run `chkdsk /f`, initialize disks, pull drives, or start a rebuild without a recovery plan.
- Move DVR recording storage away from the Drobo when a suitable replacement is ready.
