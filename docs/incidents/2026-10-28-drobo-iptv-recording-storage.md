# 2026-10-28 — IPTV recording storage outage (recovered; hardware risk open)

## Summary
Uptime Kuma monitor #31 ("IPTVX Recording Storage") went DOWN after the CIFS recording share stopped supporting filesystem operations. Service was restored without restarts/remounts. Root hardware concern remains: Drobo Gen3 Bay 1 (6 TB Seagate) is reported failed with blinking red LED; Bay 0 (6 TB Seagate) healthy. No replacement drive planned due to cost. **Incident availability recovered, data integrity and redundancy not confirmed.**

## Systems
- storage01 (192.168.1.208): Windows SMB `IPTV-Recordings` -> `G:\IPTV\Recordings`; also `CHANNELS` -> E:.
- docker (192.168.1.174): `/mnt/iptv-recordings` CIFS mount, Channels DVR (port 8089), `/usr/local/bin/check-iptvx-storage`, `iptvx-storage-monitor.timer`, Uptime Kuma (port 3001).
- fastchannels01 (192.168.1.228): upstream feeds, unaffected.

## Evidence / timeline (local PDT, Oct 28)
- ~14:29: Windows NTFS Event 50: delayed write failures including `G:\$BitMap` and `$Mft::$BITMAP`; Event 140 transaction-log flush failures, indicating possible corruption.
- ~14:34: Windows disk Event 51 and 153 I/O/retry errors; Event 140 identified Drobo Gen3 USB device and a device-not-present failure.
- Linux CIFS kernel reported `Close unmatched open for MID`; `mktemp` and `df` on recording mount failed with Input/output error; health-check exit 2.
- SMB port 445 reachable and `/mnt/movies` (E: `CHANNELS`) healthy, isolating issue to G:/recording share.
- Initially `Get-Volume -DriveLetter G` failed and `G:\IPTV\Recordings` was missing; the share remained listed as Online, but path unavailable.
- Drobo USB/PnP and Disk 3 subsequently appeared Online/Healthy; Drobo Dashboard showed Bay 1 failed (blinking red), Bay 0 healthy.
- G: returned: `Get-Volume` NTFS Healthy/OK, `Test-Path 'G:\IPTV\Recordings'` True. Reported 64 TB is virtual Drobo capacity, not physical usable capacity.
- Linux `ls -ld /mnt/iptv-recordings` succeeded; `df` showed 64T virtual, ~1.3T used; `check-iptvx-storage` returned `OK: IPTVX recording share mounted and writable`, exit 0. Uptime Kuma monitor #31 returned green.

## Current status
- **Service outage resolved**: mount, read/write smoke test, push monitoring healthy.
- **Hardware issue unresolved**: Bay 1 failed, remaining disk protection uncertain/at risk.
- **Integrity unverified**: NTFS write failures could have affected recordings or metadata; `Healthy` in Windows is not a filesystem integrity audit.
- No disk replacement authorized; user prefers not to buy a $200+ replacement for aging USB Drobo.

## Follow-up — Oct 29
1. Recheck Drobo overall protection state, Bay LEDs, G: access and fresh System disk/NTFS errors.
2. Verify and prioritize backup/copy of irreplaceable recordings to other healthy storage; avoid bulk operations if device unstable.
3. Inspect filesystem non-destructively *after* important data is protected; do not run `chkdsk /f`, initialize, reformat, pull disks, or start rebuild without a recovery plan.
4. Review available destination capacity and migrate Channels DVR recording target away from Drobo if appropriate.
5. Verify monitor #31 and document final disposition.

> Caution: A healthy Windows volume status and a passing write smoke test do not establish full data integrity or redundancy.
