# FastChannels: playlist imports and logo-cache hang — October 28, 2026

**FastChannels:** `fastchannels01` — `192.168.1.228:5523`  
**Playlist server:** `192.168.1.228:8765`

## The problem

I was importing two trimmed Streameast playlists into FastChannels:

- **FC STe01:** 700 channels
- **FC STe02:** 69 channels at the time

At first neither playlist would load. FastChannels reported a connection error because nothing was listening on port 8765, where I'd put the M3U and EPG files.

Once the HTTP server was running, the playlists imported, but FC STe01 got stuck at the very end of the logo-cache job. It showed 650/651 images on one run, then 273/274 after a restart. The scrape never finished, so later jobs couldn't proceed.

## What I found

The local files were in `/home/jatkins/iptv-imports/`:

```text
streameast_pruned_part01.m3u
streameast_pruned_part02.m3u
streameast_pruned_epg.xml.gz
```

I set up `iptv-imports-http.service` to serve that directory on port 8765, rather than relying on a Python HTTP server started in a terminal.

For the logo problem, I initially removed 388 suspect `iptv4.net` logo entries. That reduced the failed downloads but **didn't fix the hang**.

The remaining uncached “logo” turned out to be a **live MPEG-TS video URL** accidentally placed in the playlist's `tvg-logo` field. FastChannels was trying to download a never-ending video stream as if it were an image. Its image-fetch code used a 10-second inactivity timeout, which doesn't stop a stream that keeps sending data.

## The fix

I backed up the playlist, removed the bad `tvg-logo` value, and left the channel entry and actual playback URL alone. Then I restarted FastChannels to clear the stuck job and ran the imports again.

Results:

| Source | Channels | Logo-cache result |
| --- | ---: | --- |
| FC STe01 | 700 | Complete; 273 already fresh, 0 failed |
| FC STe02 | 69 | Complete; 30 already fresh, 37 failed |

Both sources showed **Complete** in FastChannels. The 37 failed logos on STe02 did not prevent the import.

## Useful checks

Run on `fastchannels01`:

```bash
systemctl status iptv-imports-http.service
systemctl is-enabled iptv-imports-http.service

curl -f -o /dev/null \
  http://192.168.1.228:8765/streameast_pruned_part01.m3u

sudo docker logs --since 10m fastchannels 2>&1 |
  grep -E 'm3u_fc-ste0[12]|pre-warm'
```

I also set up an Uptime Kuma monitor on `docker (192.168.1.174:3001)` for the playlist HTTP server. It was green on October 28; I did not separately verify notification delivery.

## Still worth checking

- FC STe02 had 37 failed logo downloads. They don't stop the source from working, but some artwork may be missing.
- The removed `iptv4.net` logos weren't the cause of the final hang. Only restore useful ones after checking their URLs.
- EPG coverage needs its own check; a completed playlist import doesn't mean the guide is complete.
- A future image-cache fix should reject non-image responses and cap total download size/time. That's not something I changed here.
