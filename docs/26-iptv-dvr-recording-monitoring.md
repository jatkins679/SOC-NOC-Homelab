# IPTV DVR and Recording Storage Monitoring

## Overview

I added a small IPTV DVR system to the homelab so I can schedule television recordings from Apple TV, keep the files on my Windows storage server, and monitor whether the recording storage is actually usable.

The main components are IPTVX on Apple TV, Dispatcharr for playlist management, IPTVX Media Server on my Debian Docker VM, and an SMB share on `storage01`.

This took more troubleshooting than I expected. Some recordings initially stopped and reconnected repeatedly, and the in-progress preview buffered without displaying video. Both problems were resolved by adjusting how Dispatcharr delivers streams and how IPTVX invokes FFmpeg.

**Project status (October 24, 2026):** Recording, scheduled recording, in-progress playback, startup after a normal VM reboot, and automated storage monitoring have been tested. Recovery from an actual SMB outage has not been tested.

---

## 1. Architecture

The recording path is:

```text
Apple TV / IPTVX
       |  schedule or preview
       v
IPTVX Media Server
docker (192.168.1.174:9090)
       |           ^
       |           | channel stream / redirect
       |       Dispatcharr
       |       docker (192.168.1.174:9191)
       |           |
       |       Original IPTV source
       |
       | FFmpeg writes MPEG-TS
       v
/mnt/iptv-recordings  (CIFS / SMB 3.1.1)
       |
       v
storage01 (192.168.1.208)
G:\IPTV\Recordings
```

Dispatcharr organizes the channels and their guide information. I use its **Redirect** stream profile, rather than having Dispatcharr's FFmpeg process the stream again. IPTVX Media Server handles recording and playback.

| Component | Recorded configuration |
|---|---|
| Apple TV client | IPTVX |
| Playlist manager | Dispatcharr, Redirect stream profile |
| Media server | IPTVX Media Server 1.0, build 37 |
| Debian VM | `docker`, `192.168.1.174` |
| Media server port | TCP 9090 |
| Dispatcharr port | TCP 9191 |
| FFmpeg | Debian FFmpeg 7.1.5 with local wrapper |
| Storage server | `storage01`, `192.168.1.208` |
| Windows folder | `G:\IPTV\Recordings` |
| SMB mount | `/mnt/iptv-recordings` |

These are recorded lab addresses, not a recommendation to publish the services to the Internet.

---

## 2. SMB Recording Storage

I created a dedicated SMB share, `IPTV-Recordings`, on the Windows storage server and a non-administrator local account used only for DVR storage.

The Debian VM mounts:

```text
//192.168.1.208/IPTV-Recordings
    -> /mnt/iptv-recordings
```

The share is configured in `/etc/fstab` with SMB 3.1.1 and a systemd automount. Credentials live in a root-owned file outside the repository.

I verify the actual CIFS layer, not just the automount placeholder:

```bash
findmnt --kernel -T /mnt/iptv-recordings -t cifs \
  -o TARGET,SOURCE,FSTYPE
```

The underlying local directory is `root:root`, mode `0755`. I inspected it separately from the mounted share and confirmed the `jatkins` service account cannot write to it. If the share is unmounted, that prevents ordinary recording writes from silently filling the Debian VM's much smaller system disk.

It does **not** guarantee recovery if the SMB connection fails while CIFS remains mounted: I/O can stall or fail even when the mount still appears present.

---

## 3. IPTVX Media Server as a systemd Service

The server is installed under:

```text
/home/jatkins/iptvx-install/iptvx-server/
```

The `iptvx-server.service` unit starts its `start.sh` script as `jatkins`, after the network and recording mount are available. An `ExecStartPre` check verifies CIFS is mounted before startup.

Selected unit directives:

```ini
[Unit]
Description=IPTVX Media Server
Wants=network-online.target
After=network-online.target
RequiresMountsFor=/mnt/iptv-recordings

[Service]
Type=simple
User=jatkins
Group=jatkins
WorkingDirectory=/home/jatkins/iptvx-install/iptvx-server
ExecStartPre=/usr/bin/findmnt --kernel --mountpoint /mnt/iptv-recordings --types cifs --noheadings
ExecStart=/home/jatkins/iptvx-install/iptvx-server/start.sh
Restart=on-failure
RestartSec=10
NoNewPrivileges=true

[Install]
WantedBy=multi-user.target
```

The server configuration uses the SMB mount as `RECORDINGS_DIR` and points `FFMPEG_PATH` at a custom wrapper. The API token and the actual configuration file contents are deliberately excluded here.

Useful checks:

```bash
systemctl is-enabled iptvx-server
systemctl is-active iptvx-server
sudo systemctl status iptvx-server --no-pager
```

After rebooting `docker`, I confirmed the IPTVX service, systemd automount, and CIFS mount had all returned successfully.

---

## 4. Recording and Preview Troubleshooting

### Dispatcharr was reconnecting

Direct playlist streams could record, but recordings through Dispatcharr repeatedly switched to **Reconnecting**.

Dispatcharr's default stream profile was **FFmpeg**, with MPEG-TS output. Changing the default profile to **Redirect** stopped the extra stream-processing hop and allowed IPTVX to record through Dispatcharr in the tested configuration.

I kept MPEG-TS output selected.

### FFmpeg itself also needed adjustment

IPTVX bundles an FFmpeg binary, but I encountered a segmentation fault while testing a stream. I installed the Debian FFmpeg package and set IPTVX to invoke it through `/usr/local/bin/iptvx-ffmpeg`.

The wrapper removes IPTVX's aggressive HTTP reconnect arguments. For HTTP(S) input URLs ending in `.m3u8` (before any query string), it adds `-extension_picky 0` to allow the HLS input we tested.

The working wrapper is:

```bash
#!/usr/bin/env bash

args=()

while (($#)); do
    case "$1" in
        -reconnect|-reconnect_streamed|-reconnect_at_eof|-reconnect_delay_max)
            shift
            if (($#)); then shift; fi
            ;;
        -i)
            if (($# >= 2)) && [[ "$2" =~ ^https?:// ]] &&
               [[ "${2%%\?*}" == *.m3u8 ]]; then
                args+=("-extension_picky" "0")
            fi
            args+=("$1")
            shift
            ;;
        *)
            args+=("$1")
            shift
            ;;
    esac
done

exec /usr/bin/ffmpeg "${args[@]}"
```

This is a workaround for the observed streams, not a general replacement for FFmpeg's built-in reconnect handling.

### The preview would only buffer

While a recording was active, IPTVX showed an option to start a preview, but playback remained stuck buffering.

The journal revealed repeated snapshot failures:

```text
snapshot capture attempt failed
Option extension_picky not found
```

The earlier wrapper was injecting the HLS-only `extension_picky` option before **every** `-i`, including local MPEG-TS recording files. That broke the FFmpeg jobs IPTVX launched to generate preview thumbnails.

I corrected the wrapper to add the option only for HTTP(S) `.m3u8` inputs. As a direct test, I generated a JPEG from a saved `.ts` recording. After restarting IPTVX, preview playback worked while the recording continued without reconnecting.

---

## 5. Uptime Kuma Monitoring

The DVR has three separate monitors in Uptime Kuma, running on `docker` at `192.168.1.174:3001`.

| Monitor | Type | What it proves |
|---|---|---|
| IPTVX Media Server | HTTP(S), port 9090 | Web interface responds |
| storage01 - SMB | TCP port 445 | SMB listener is reachable |
| IPTVX - Recording Storage | Push | CIFS is mounted and a read/write check succeeds |

The first two checks alone cannot confirm that the recording directory is writable.

For the third monitor, I created:

- `/usr/local/bin/check-iptvx-storage` — checks for a real CIFS mount, creates a small temporary file as `jatkins`, writes and reads it back, then removes it.
- `/usr/local/bin/report-iptvx-storage` — runs the check with a 20-second timeout and sends an UP/DOWN heartbeat to Uptime Kuma.
- `/etc/iptvx-storage-monitor.env` — root-owned configuration containing the private Push URL; **not committed**.
- `iptvx-storage-monitor.service` — one-shot systemd service running the reporter.
- `iptvx-storage-monitor.timer` — schedules subsequent checks at 60-second intervals.

The reporter's basic status logic is:

```bash
RESULT=$(timeout 20s runuser -u jatkins -- \
    /usr/local/bin/check-iptvx-storage 2>&1)
RC=$?

if [[ $RC -eq 0 ]]; then
    STATUS="up"
    MESSAGE="Storage mounted and writable"
else
    STATUS="down"
    MESSAGE="Storage check failed (exit $RC)"
fi
```

It then sends `status` and `msg` as URL-encoded query parameters to the private Uptime Kuma Push URL using `curl --fail --max-time 10`.

Selected timer settings:

```ini
[Timer]
OnBootSec=60s
OnUnitActiveSec=60s
AccuracySec=1s
Unit=iptvx-storage-monitor.service
```

When I first enabled the timer, its next execution showed `n/a`. Running the one-shot service once established its activation time; afterward the timer displayed the next run correctly. I subsequently confirmed an automatic successful execution.

Checks:

```bash
sudo systemctl status iptvx-storage-monitor.service --no-pager
systemctl list-timers iptvx-storage-monitor.timer --no-pager
sudo journalctl -u iptvx-storage-monitor.service -n 20 --no-pager
```

A successful push proves the share was usable at the time of the check. It does not verify the content of every recording. A kernel task stuck in uninterruptible CIFS I/O may also outlast the userspace timeout.

---

## 6. Problems Encountered

| Symptom | Cause or finding | Resolution |
|---|---|---|
| Bundled FFmpeg crashed | Segmentation fault observed on a tested stream | Use Debian FFmpeg via wrapper |
| IPTVX reconnected during recording | Connection behavior and, with Dispatcharr, additional stream processing | Remove selected reconnect flags; use Dispatcharr Redirect profile |
| In-progress preview buffered indefinitely | HLS-only option incorrectly applied to local `.ts` files | Limit `-extension_picky 0` to HTTP(S) `.m3u8` inputs |
| Snapshot generation failed | Same FFmpeg argument error | Correct wrapper and verify JPEG extraction |
| Storage Push monitor initially returned 404 | Push URL did not resolve to the expected Kuma monitor | Replace with URL generated by the Push monitor |
| Push reporter failed with shell syntax error | Unmatched quote in private environment file | Correct quoting and verify with `bash -n` |
| Timer initially displayed `n/a` | No initial one-shot service activation available | Run service once and verify subsequent timer executions |

---

## 7. Remaining Work

- [x] Confirm IPTVX saves real video to the Windows SMB share.
- [x] Confirm scheduled recording can be initiated from Apple TV.
- [x] Record through Dispatcharr using Redirect mode.
- [x] Preview an unfinished recording while recording continues.
- [x] Verify the server and CIFS mount return after a normal Debian reboot.
- [x] Monitor IPTVX HTTP availability, SMB connectivity, and actual share read/write capability.
- [ ] Test a real SMB/network interruption and document behavior and recovery.
- [ ] Consider checking recording integrity and failures separately from server uptime.

The completed work gives me a practical example of multi-service troubleshooting: following a stream through a proxy and recorder, reading systemd and FFmpeg logs, dealing with SMB storage, and turning a one-off fix into an ongoing monitor.

**Security note:** This public document deliberately excludes provider stream URLs, embedded credentials, SMB passwords, the IPTVX token, the Uptime Kuma Push token, and private environment file contents.
