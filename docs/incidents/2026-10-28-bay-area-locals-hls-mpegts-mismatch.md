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

A live stream can exceed a size or time limit and cause `curl` to exit nonzero even after valid transport-stream bytes have been captured. 

## Resolution

In Channels DVR, edited the **Bay Area Locals** custom source and changed its stream format from **HLS** to **MPEG-TS (TS)**. No change to the upstream M3U entries, FastChannels sources, channel IDs, or XMLTV guide mappings was required.

## Verification

The operator confirmed: **“they're all working now with TS and not HLS.”** This verifies restored playback for the affected Bay Area Locals lineup on October 28, 2026.

## Operational lesson

A `.m3u8`-looking endpoint or an HTTP 302 does not establish that its final payload is HLS. Trace redirects and inspect the final response content type and first bytes before changing playlists or rebuilding EPG mappings. Source-level playback format settings in Channels DVR must match the actual media returned. If a source mixes MPEG-TS and HLS streams, verify compatibility before changing the format for the entire lineup.

See also [Channels DVR and FastChannels Media Services](../27-channels-dvr-fastchannels.md).

## Recurrence during clean Channels DVR source rebuild — 2026-10-30

**Status:** Resolved; all six Bay Area Local channels reported playing in the Channels DVR web player after correction.

### Context and impact

The operator removed the accumulated Channels DVR custom M3U sources and rebuilt from FastChannels on `fastchannels01 (192.168.1.228:5523)`. The `FC Bay Area Local` feed was configured in **Separate playlists** guide mode rather than a mixed XMLTV/Gracenote playlist. Channels DVR (`docker, 192.168.1.174:8089`) registered:

- `M3U-FastChannelsFCBayAreaLocal` → `XMLTV-FastChannelsFCBayAreaLocal` (five standard XMLTV channels)
- `M3U-FastChannelsFCBayAreaLocalGracenote` → `X-M3U` (one Gracenote channel)

Guide entries and logos appeared for all six stations: **KGO, KRON, KPIX, KTVU, KNTV, and KQED**. However, the five standard/XMLTV channels failed in the Channels DVR web player with network or unrecognized-format errors. **KQED (Gracenote) played normally.** This separated successful EPG ingestion from unsuccessful stream playback.

### Diagnostic evidence

The standard FastChannels M3U exposed the KGO stream as:

```text
http://192.168.1.228:5523/play/m3u_fc-ste01/465959.m3u8
```

A `curl -L` check from `docker` showed FastChannels responding with HTTP **302**, redirecting to an upstream **.ts** resource; a subsequent upstream redirect returned HTTP **200**. The captured payload was positively identified by `file` as **MPEG transport stream data**. About **16.9 MB** of live stream data arrived before the 15-second request timed out, so the timeout was not evidence of a dead stream. Private provider URL paths and authentication tokens are intentionally excluded.

### Corrective action

In Channels DVR's **FastChannels FC Bay Area Local** standard five-channel M3U source, changed **Stream Format** from **HLS** to **MPEG-TS**. The working Gracenote source was left unchanged. No EPG remapping was needed.

### Verification and preventive guidance

The operator then confirmed: **“they all play now”**. Combined with visible guide entries for all six stations, this confirms both EPG coverage and web-player playback after the format correction. This is independent of any future recording reliability test.

For future FastChannels → Channels DVR imports:

1. Keep matching standard XMLTV and Gracenote guide sources separately when needed.
2. Test representative **actual playback payloads**, not just M3U URL extensions or successful guide imports.
3. Use **MPEG-TS** for a Channels DVR source whose redirected streams deliver TS, even if the FastChannels endpoint ends with `.m3u8`.
4. Verify playback and guide listings before promoting the feed into the production lineup. Avoid blindly applying MPEG-TS to unrelated feeds that may truly deliver HLS.
