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

## Freshness monitoring — installed 2026-10-28

The **FastChannels EPG Freshness** Uptime Kuma Push monitor (#35) was created on docker (192.168.1.174:3001). A separate Pushover notification integration was tested successfully to an iPhone. The user confirmed monitor #35 is **green/up** on 2026-10-28. The user verified that `Homelab Pushover Alerts` is enabled on monitor #35 (Slack disabled).

On fastchannels01 (192.168.1.228):

- Checker: `/usr/local/sbin/check-fastchannels-epg` (root:root, 0700).
- Private Uptime Kuma push URL: `/etc/fastchannels-epg/kuma-push.url` (root:root, 0600), in root-only directory.
- Service: `/etc/systemd/system/fastchannels-epg-health.service`, root-run `Type=oneshot`, `ExecStart=/usr/local/sbin/check-fastchannels-epg`, 120-second timeout.
- Timer: `/etc/systemd/system/fastchannels-epg-health.timer`, `OnCalendar=*:0/5`, `Persistent=true`, enabled and active.
- Healthy criteria: published `streameast_pruned_epg.xml.gz` exists, its mtime is at most **15 hours old**, is parseable gzip/XML, and has at least **400 unique EPG IDs currently airing**. The checker also counts programs with future stop times.
- Checker pushes `status=up` or `status=down` to Kuma. Missing pushes are also detected by Kuma according to its Push monitor heartbeat/grace settings. File age alone is not sufficient evidence of live EPG coverage.

**Verified manual test:** exit status 0; response `UP: EPG healthy: 620 current IDs, 9502 upcoming programs, age 0.6h`. The push request completed without an error. The systemd timer was verified `enabled` and `active`. Subsequently, its journal confirmed **five consecutive successful automatic executions** at 20:15:01, 20:20:08, 20:25:11, 20:30:08 and 20:35:08 UTC on 2026-10-28, all reporting `UP` without errors. The latest check had 620 currently airing EPG IDs, 9,273 upcoming programs, and a 1.0-hour-old guide. The next scheduled run was 20:40 UTC. The user also confirmed monitor #35 green. On 2026-10-28, a simulated `status=down` Push report with retries set to 0 successfully delivered a Pushover DOWN alert to the iPhone. Earlier attempts with 2 retries entered Pending rather than causing an alert. Restore an UP heartbeat afterward with `sudo /usr/local/sbin/check-fastchannels-epg` (the user subsequently confirmed monitor #35 returned green).

```bash
systemctl is-enabled fastchannels-epg-health.timer
systemctl is-active fastchannels-epg-health.timer
systemctl list-timers --all fastchannels-epg-health.timer
sudo journalctl -u fastchannels-epg-health.service -n 30 --no-pager
systemctl show fastchannels-epg-health.service -p Result -p ExecMainStatus
sudo /usr/local/sbin/check-fastchannels-epg
```

**Notification verified:** `Homelab Pushover Alerts` is enabled on monitor #35. The user confirmed a real Pushover iPhone alert triggered by a simulated DOWN Push report after setting monitor retries to 0. **Recovery confirmed:** the user reported monitor #35 green again after the DOWN alert test. Whether a separate recovery notification was received was not checked. Consider an isolated failure test that does not modify the production guide or expose the private Push URL. The existing 8765 HTTP monitor separately checks that playlist hosting is reachable; it cannot detect guide staleness.

## Security

Keep provider URL and Uptime Kuma push URL in root-only files (0600; directories 0700). Never echo either secret in shell output, send them to GitHub, or include them in terminal screenshots. Do not expose these files through the `/home/jatkins/iptv-imports` public HTTP document root.
