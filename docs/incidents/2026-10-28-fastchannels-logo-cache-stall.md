# Incident: FastChannels M3U imports and logo-cache stall (2026-10-28)

## Summary

Two user-defined M3U sources in FastChannels on `fastchannels01 (192.168.1.228)` experienced failures: FC STe01 (700 channels) and FC STe02 (69 channels). The initial playlist fetch failures were caused by the local HTTP server on TCP 8765 not running. After restoring the server, one malformed `tvg-logo` entry caused the image pre-warm job to hang at effectively 100%, preventing subsequent scrape jobs from progressing.

**Outcome:** Both sources returned to **Complete**. 769 channels were imported in total. FC STe02's last image-cache run finished with 37 failed image fetches; these did not block completion.

## Components

- `fastchannels01 (192.168.1.228:5523)`: FastChannels v5.7.2 running in Docker.
- `fastchannels01 (192.168.1.228:8765)`: locally hosted M3U and XMLTV artifacts in `/home/jatkins/iptv-imports/`.
- `docker (192.168.1.174:3001)`: Uptime Kuma, where a new HTTP monitor is recommended (not yet confirmed configured).

## Symptoms and investigation

1. Both sources reported `Playlist could not be fetched ... ConnectionError`. The host had no listener on TCP 8765.
2. The playlist directory contained `streameast_pruned_part01.m3u`, `streameast_pruned_part02.m3u`, and `streameast_pruned_epg.xml.gz`.
3. The manually launched Python HTTP server responded with HTTP 200. A dedicated systemd service was created to survive SSH disconnects and VM reboots.
4. A subsequent FC STe01 scrape imported all 700 channels in approximately 0.9 seconds, but logo pre-warming stopped at 650/651. Earlier fetch logs showed 300 failures, prompting investigation of logo hosts. Removing 388 suspect `iptv4.net` logo references reduced noise but **did not by itself fix the hang**.
5. Following a controlled restart, FC STe01 again stopped at 273/274: 273 cached and one uncached URL.
6. Inspecting `/app/app/routes/images.py` in the FastChannels container showed `requests.Session.get(url, timeout=10)`, followed by `r.content`. This is an inactivity timeout, **not a total-download deadline**.
7. Comparison of playlist logo URL MD5 hashes with `/data/logo_cache/logos` identified the only uncached URL as an HTTP `/live/.../*.ts` **video-stream URL incorrectly placed in `tvg-logo`**. The full upstream URL is intentionally omitted because it contained account-like identifiers.
8. A separate bounded `curl` request to that URL had returned `video/mp2t` and continued transferring until the client deadline. Downloading it as a logo could therefore prevent the pre-warm task from finishing.

## Resolution

- Created and enabled `/etc/systemd/system/iptv-imports-http.service` to serve `/home/jatkins/iptv-imports` on TCP 8765 as user `jatkins`.
- Backed up FC STe01 before each targeted logo edit. Removed the one malformed `.ts` logo reference while leaving the `#EXTINF` entry, the actual stream URL, channel identifiers, and other metadata intact.
- Restarted the persistent-volume-backed FastChannels Docker container to clear the already-running logo fetch.
- Rescraped FC STe01: **700 channels, pre-warm done: 0 cached, 273 already fresh, 0 failed**.
- Rescraped FC STe02: **69 channels, pre-warm done: 0 cached, 30 already fresh, 37 failed**. Source completed; failed logos remain a nonblocking follow-up.
- Verified both sources displayed green **Complete** in the FastChannels UI.

## Service maintenance

```bash
systemctl status iptv-imports-http.service
systemctl is-enabled iptv-imports-http.service
curl -f -o /dev/null http://192.168.1.228:8765/streameast_pruned_part01.m3u
sudo docker logs --since 10m fastchannels 2>&1 | grep -E 'm3u_fc-ste0[12]|pre-warm'
```

## Follow-up

- [x] Uptime Kuma on `docker (192.168.1.174:3001)`: the playlist-server monitor has been created and the user confirmed it is **green/up** on 2026-10-28. Suggested configuration: HTTP(s) GET, `http://192.168.1.228:8765/streameast_pruned_part01.m3u`, 60-second interval, 3 retries, accepted status codes 200–299. Notification association was not separately verified.
- [ ] Investigate FC STe02's 37 failed image downloads if artwork is important.
- [ ] Review EPG coverage separately: FastChannels source scrape logs reported very limited guide programs. This is not a finding about every EPG integrated into Channels DVR.
- [ ] Consider guarding image pre-warm against non-image content and unbounded streaming responses (content-type validation, streaming byte cap, and full-request deadline).
- [ ] Optionally restore legitimate `iptv4.net` logos from the saved pre-cleanup backup only after validating individual URLs; they were removed during diagnosis and were **not** confirmed to be the cause of the final stall.
