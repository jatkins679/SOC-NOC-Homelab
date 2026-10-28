# FastChannels StreamEast EPG refresh and freshness monitoring

**Date:** 2026-10-28  
**Host:** fastchannels01 (192.168.1.228)  
**FastChannels:** Docker on port 5523; local playlist/XMLTV HTTP server on port 8765  
**Downstream:** Channels DVR on docker (192.168.1.174:8089); Uptime Kuma on docker (192.168.1.174:3001)

## Background and verified result

The original pruned XMLTV guide had almost entirely expired programming. A fresh provider download was filtered to the IDs in the two curated playlists. The playlist sources remained FC STe01 (700 channels, source ID 54, m3u_fc-ste01) and FC STe02 (69 channels, source ID 55, m3u_fc-ste02), for 769 entries total.

The 2026-10-28 19:32 UTC end-to-end test confirmed:

- Provider feed downloaded, validated and republished successfully.
- 10,049 future/current programmes in filtered guide; 620 distinct matching EPG IDs, all 620 airing at the time of testing.
- Systemd unit returned Result=success and ExecMainStatus=0.
- Both source-trigger API requests returned queued; worker logs **subsequently confirmed completion**:
  - FC STe01: 700 channels, 11,099 imported programme entries, covering 626 channels.
  - FC STe02: 69 channels, 455 imported programme entries, covering 67 channels.
- Both source records reported enabled=true and last_error=null.
- Worker log briefly stated that XML regeneration was deferred while the second scrape was pending. This is not an error.

Programme counts imported into individual playlist sources can exceed the number of unique raw XMLTV programme elements because multiple playlist entries can share EPG IDs.

## Running services and paths

- Provider URL (contains credentials): `/etc/fastchannels-epg/provider.url`, **root:root 0600** inside root-owned directory, **never commit or paste its contents**.
- Refresh script: `/usr/local/sbin/refresh-fastchannels-epg` (root:root 0700).
- Trigger script: `/usr/local/sbin/trigger-fastchannels-epg` (root:root 0700).
- Unit: `/etc/systemd/system/fastchannels-epg-refresh.service` (`Type=oneshot`, `ExecStart=/usr/local/sbin/refresh-fastchannels-epg`, `TimeoutStartSec=300`).
- Drop-in: `/etc/systemd/system/fastchannels-epg-refresh.service.d/post-refresh.conf` (`ExecStartPost=/usr/local/sbin/trigger-fastchannels-epg`).
- Timer: `/etc/systemd/system/fastchannels-epg-refresh.timer`, `OnCalendar=*-*-* 00,12:00:00`, `Persistent=true`, `RandomizedDelaySec=300`.
- Curated inputs: `/home/jatkins/iptv-imports/streameast_pruned_part01.m3u`, `streameast_pruned_part02.m3u`.
- Published output: `/home/jatkins/iptv-imports/streameast_pruned_epg.xml.gz`.
- Last-good backup: `/home/jatkins/iptv-imports/streameast_pruned_epg.last-good.xml.gz`.
- Local HTTP guide URL: `http://192.168.1.228:8765/streameast_pruned_epg.xml.gz`.

The script downloads provider XMLTV over HTTPS, parses guide times, filters to matching playlist `tvg-id` values, validates minimum counts (3,000 programmes, 550 EPG IDs, 400 currently airing IDs), checks produced gzip XML, backs up the prior guide, and atomically publishes the replacement. Failed validation preserves the previous published guide. The provider source URL must never appear in committed scripts, issue comments or logs.

After successful publication only, systemd triggers the two local FastChannels source-run API calls. The API queues **full** scrapes, not EPG-only scrapes, and queue acceptance is distinct from worker completion. FastChannels has a 720-minute default playlist scrape interval; the post-refresh trigger avoids timing drift between independent 12-hour schedules.

The VM uses UTC; the timer runs around 00:00 and 12:00 UTC, with up to five minutes randomized delay. **These are 17:00 and 05:00 Pacific during PDT**, and 16:00 and 04:00 during PST.

## Maintenance and verification

```bash
systemctl list-timers --all fastchannels-epg-refresh.timer
systemctl cat fastchannels-epg-refresh.service
sudo systemctl start fastchannels-epg-refresh.service
systemctl show fastchannels-epg-refresh.service -p Result -p ExecMainStatus
sudo journalctl -u fastchannels-epg-refresh.service -n 50 --no-pager
sudo docker logs --since 10m fastchannels 2>&1 | grep -E 'm3u_fc-ste01|m3u_fc-ste02'
gzip -t /home/jatkins/iptv-imports/streameast_pruned_epg.xml.gz
```

Manual start queues both sources when successful; inspect worker log for each `Scrape complete` message and check `last_error` via `GET /api/sources`. An inactive one-shot service between executions is normal. Rollback requires preserving/validating the backup before replacing the published guide, then triggering the two sources to reimport it.

## Freshness monitoring — follow-up configuration

**Recommended implementation (not yet confirmed installed):** Create a Uptime Kuma Push monitor called `FastChannels StreamEast EPG freshness`, with a 5-minute push interval and grace period. Store its private push URL as a root-only file on fastchannels01 (not in GitHub), then run a root-owned, systemd-timer-driven checker every five minutes. A checker should report UP only when the published XMLTV is readable and its **file modification time is less than 15 hours old**; also verify that recent/future programmes exist to avoid declaring an outdated provider guide healthy. Report DOWN on a stale/invalid guide; missing pushes should also cause a Kuma alert. Test a simulated stale guide without altering the production file, and confirm Kuma shows green after deployment.

The existing playlist HTTP monitor remains useful for port 8765 reachability but is **not** an EPG freshness monitor. Monitor creation, push URL setup, checker deployment and successful green state remain to be verified. Configure notification delivery in Uptime Kuma separately.

## Security

Keep provider URL and Uptime Kuma push URL in root-only files (0600; directories 0700). Never echo either secret in shell output, send them to GitHub, or include them in terminal screenshots. Do not expose these files through the `/home/jatkins/iptv-imports` public HTTP document root.
