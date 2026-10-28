# Incident: Bay Area Locals HLS/MPEG-TS playback mismatch (2026-10-28)

**Status:** Resolved  
**Affected service:** Channels DVR custom source, Bay Area Locals  
**Systems:** `docker (192.168.1.174:8089)` — Channels DVR; `fastchannels01 (192.168.1.228:5523)` — FastChannels  
**Impact:** Bay Area Locals stations did not play in the Apple TV client.

## Symptom

Playback failed with the message:

```text
detected MPEG-TS instead of HLS playlist
```

## Investigation

1. Located KTVU and KPIX entries in the locally imported M3U playlists on `fastchannels01`. The upstream stream URLs ended in `.ts`. That suffix was only an indication; the actual response format still needed verification.
2. FastChannels logs showed requests from Channels DVR (`192.168.1.174`) to `/play/m3u_fc-ste01/466976.m3u8` for KTVU, each returning HTTP **302**.
3. The redirect's `Location` targeted the original upstream `.ts` stream rather than an HLS manifest. The private upstream URL is deliberately omitted.
4. A bounded `curl -L` check returned HTTP **200**, `Content-Type: video/mp2t`, and binary content starting with `47 40 11` (consistent with MPEG transport-stream packets). The request timed out after receiving about 18.5 MB of ongoing video; that timeout did **not** indicate the source was offline.
5. This explained the format mismatch: despite the FastChannels playback route ending in `.m3u8`, its redirect returned MPEG-TS data, not an HLS playlist starting with `#EXTM3U`.

### Useful read-only checks

Run on `fastchannels01 (192.168.1.228)`:

```bash
sudo docker logs --since 30m fastchannels 2>&1 | grep -i -C 2 'KTVU'
URL="http://192.168.1.228:5523/play/m3u_fc-ste01/466976.m3u8"
curl -sS --max-time 10 -D - -o /dev/null "$URL" | grep -Ei '^HTTP/|^Location:|^Content-Type:'
# Take care not to publish the Location header: it may contain private stream tokens.
```

For a bounded upstream content check, follow redirects while limiting the response:

```bash
curl -sSL --max-time 10 --max-filesize 1048576 -D /tmp/stream-headers.txt \
  -o /tmp/stream-check.bin "$URL"
grep -Ei '^HTTP/|^Content-Type:' /tmp/stream-headers.txt
head -c 64 /tmp/stream-check.bin | xxd
```

A live stream can exceed a size or time limit and cause `curl` to exit nonzero even after valid transport-stream bytes have been captured. Do not publish raw redirect locations or downloaded content.

## Resolution

In Channels DVR, edited the **Bay Area Locals** custom source and changed its stream format from **HLS** to **MPEG-TS (TS)**. No change to the upstream M3U entries, FastChannels sources, channel IDs, or XMLTV guide mappings was required.

## Verification

The operator confirmed: **“they're all working now with TS and not HLS.”** This verifies restored playback for the affected Bay Area Locals lineup on October 28, 2026.

## Operational lesson

A `.m3u8`-looking endpoint or an HTTP 302 does not establish that its final payload is HLS. Trace redirects and inspect the final response content type and first bytes before changing playlists or rebuilding EPG mappings. Source-level playback format settings in Channels DVR must match the actual media returned. If a source mixes MPEG-TS and HLS streams, verify compatibility before changing the format for the entire lineup.

See also [Channels DVR and FastChannels Media Services](../27-channels-dvr-fastchannels.md).
