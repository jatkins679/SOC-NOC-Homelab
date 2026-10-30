# Bay Area Locals: HLS vs. MPEG-TS

**Fixed:** October 28, 2026 (rechecked October 30)  
**Channels DVR:** docker — `192.168.1.174:8089`  
**FastChannels:** fastchannels01 — `192.168.1.228:5523`

## The problem

Bay Area Local streams were showing up in Channels DVR but wouldn't play. The original error was:

```text
detected MPEG-TS instead of HLS playlist
```

We ran into the same problem again on October 30 while rebuilding the Channels DVR M3U sources from scratch.

The new FastChannels Bay Area Local feed had six stations: KGO, KRON, KPIX, KTVU, KNTV, and KQED. All six had guide listings and logos, but only KQED played in the Channels DVR web player. The other five returned network or unsupported-format errors.

## What I found

The failing channels came from the `FC STe01` source in FastChannels. For example, KGO's playlist entry pointed to:

```text
http://192.168.1.228:5523/play/m3u_fc-ste01/465959.m3u8
```

Despite the `.m3u8` filename, that URL redirects to an MPEG-TS stream, not an HLS playlist.

I checked KGO from `docker`:

```bash
URL="http://192.168.1.228:5523/play/m3u_fc-ste01/465959.m3u8"

curl -sS -L --max-time 15 \
  -o /tmp/kgo-stream-sample "$URL"

file /tmp/kgo-stream-sample
```

Result:

```text
/tmp/kgo-stream-sample: MPEG transport stream data
```

The request timed out because it was pulling a live stream; it had already received about 17 MB of video. The stream itself was fine. Channels DVR was expecting the wrong format.

## The fix

In Channels DVR, I edited **FastChannels FC Bay Area Local** (the standard five-channel source) and changed **Stream Format** from **HLS** to **MPEG-TS**.

I left the separate Gracenote source alone because KQED was already working.

On the October 30 rebuild, the source mappings were:

| Source | Guide |
| --- | --- |
| `M3U-FastChannelsFCBayAreaLocal` | `XMLTV-FastChannelsFCBayAreaLocal` |
| `M3U-FastChannelsFCBayAreaLocalGracenote` | `X-M3U` |

After the change, all six Bay Area channels played in the Channels DVR web player, and their EPG listings were still there.

## For next time

If a FastChannels source has guide data but won't play, check the stream format before rebuilding the playlist or changing the EPG. An endpoint ending in `.m3u8` doesn't necessarily return HLS.

The October 28 issue was the same basic mismatch, and changing Channels DVR to TS fixed it then too.

See [Channels DVR and FastChannels](../27-channels-dvr-fastchannels.md).
