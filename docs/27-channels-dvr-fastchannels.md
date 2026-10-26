# Channels DVR and FastChannels Media Services

## Overview

I added Channels DVR and a separate FastChannels server to organize free live-TV sources alongside my existing IPTV services. This uses the Proxmox cluster, Docker, the Windows NAS, and Uptime Kuma monitoring.

**Status: operational and recovery-tested on October 25, 2026.** The FastChannels backup was tested by restoring it to an isolated temporary VM and verifying that its local web interface returned HTTP 200. This validates a basic application boot, not every individual channel or recording.

## Architecture

```text
Apple TV / Channels app
          |
          v
Channels DVR — docker (VM 210, pve02)
192.168.1.174:8089
      |              |
      |              +-- Dispatcharr — 192.168.1.174:9191
      |              +-- Samsung TV Plus connector — 192.168.1.174:8182
      |
      +-- FastChannels — fastchannels01 (VM 330, pve04)
                            192.168.1.228:5523
                            |
                            +-- Pluto TV
                            +-- PBS / Public Television
                            +-- Free Networks
                            +-- C-SPAN
                            +-- FreeLiveSports
                            +-- Adult Swim
                            +-- Zeam
                            +-- Local Now

Monitoring: Uptime Kuma — docker (192.168.1.174:3001)
Backups:    pve04 -> t-20-backup -> storage01 (192.168.1.208)
```

These addresses refer to the current internal homelab, not public-facing endpoints. The older IPTVX/Dispatcharr recording arrangement is documented separately in [IPTV DVR and Recording Storage Monitoring](26-iptv-dvr-recording-monitoring.md).

## System inventory

| Component | Location | Configuration |
|---|---|---|
| Channels DVR | `docker`, VM 210 on `pve02` | `192.168.1.174:8089` |
| Dispatcharr | Docker on `docker` | `192.168.1.174:9191` |
| Samsung TV Plus connector | Docker on `docker` | `192.168.1.174:8182` |
| FastChannels | `fastchannels01`, VM 330 on `pve04` | `192.168.1.228:5523` |
| Uptime Kuma | Docker on `docker` | `192.168.1.174:3001` |
| Shared backup storage | `storage01` | `192.168.1.208`, CIFS `t-20-backup` |

VM 330 runs Ubuntu Server 24.04, two vCPUs, 4 GiB RAM, and a 24 GiB virtual disk. Its Docker Compose project is in `/home/jatkins/fastchannels`. The container has a 2 GiB memory limit, a combined 3 GiB memory-and-swap limit, and two persistent named volumes for application data and Android Debug Bridge data.

The container restart policy is `unless-stopped`; Proxmox has `onboot: 1` and `startup: order=3,up=30` for VM 330. The latter waits 30 seconds **after** VM 330 starts before moving to the next VM; it does not delay VM 330's own boot.

## Source feeds and numbering

I use dedicated FastChannels feeds rather than importing the Default feed, which would mix sources and can create duplicates. Each feed has a single source filter and uses the mixed playlist/guide option supported by this installation.

| Source | Starting channel number | Notes |
|---|---:|---|
| Pluto TV | 200000 | Dedicated Pluto feed |
| PBS / Public Television | 10000 | Some PBS entries are DRM-protected |
| Free Networks | 15000 | Audit identified one dead and one HTTP 403 stream during setup |
| C-SPAN | 20000 | Public affairs/event streams; not necessarily equivalent to cable lineups |
| FreeLiveSports | 25000 | Sports-related FAST programming |
| Adult Swim | 30000 | Themed programming streams |
| Zeam | 35000 | 412 channels and 118,757 EPG records discovered in the documented scrape |
| Local Now | 40000 | Local/regional news, weather, entertainment |

These numbers are the feed starting values used or prescribed during setup; verify current assignments in the FastChannels UI before reusing a range. User testing confirmed playback and EPG for Local Now and the other integrated sources. A healthy source feed does not guarantee that every individual channel will remain available. I do not attempt to bypass DRM or HTTP 403 restrictions.

## Operations

FastChannels admin: `http://192.168.1.228:5523/admin/`

Channels DVR: `http://192.168.1.174:8089/`

Useful read-only checks on **fastchannels01 (192.168.1.228)**:

```bash
cd ~/fastchannels
sudo docker compose ps
sudo docker inspect fastchannels --format 'Restart={{.HostConfig.RestartPolicy.Name}} Status={{.State.Status}}'
sudo docker stats --no-stream fastchannels
free -h
df -h /
curl -sS -o /dev/null -w '%{http_code}\n' http://127.0.0.1:5523/admin/
```

To inspect source status, channel review, audit results, feeds and their guide modes, use the FastChannels admin UI. The live TV clients connect to Channels DVR; individual FastChannels source feeds are added there through their dedicated **Add to Channels DVR** actions.

### Resource investigation

After importing several providers, Docker reported approximately 533 MiB RAM in use and 0.45% CPU during an idle check. A previous CPU reading around 101% occurred while the system was performing source work, and the host later showed a low load average. The root filesystem was 63% used (approximately 7.8 GiB available). Docker reported a large FastChannels image (7.6 GB) and roughly 634 MB of persistent data. Image and filesystem figures have different accounting methods and should not be added directly.

The immediate decision was to preserve the installation and monitor free disk space, rather than run a destructive Docker prune.

### Uptime Kuma

Uptime Kuma on `docker (192.168.1.174:3001)` has an HTTP monitor named `fastchannels01 - FastChannels` for:

```text
http://192.168.1.228:5523/admin/
```

The monitor uses a 60-second interval, three retries, and a 10-second timeout; it was confirmed green. This proves the web UI responds, **not** that every channel is playable or EPG data is current.

## Backup configuration

VM 330 is included in the existing **all-VM** backup schedule for pve04:

| Setting | Value |
|---|---|
| Schedule | Daily 06:00 |
| Mode | Snapshot |
| Compression | ZSTD |
| Storage | `t-20-backup` (SMB `ProxmoxBackups` on `storage01`) |
| Retention | `keep-last=7,keep-weekly=4,keep-monthly=3` |

The first verified backup was:

```text
t-20-backup:backup/vzdump-qemu-330-2026_10_25-18_23_19.vma.zst
```

The archive was approximately 6.8 GB and passed `zstd -t`. Snapshot backups can be crash-consistent rather than fully application-consistent when the app is writing data.

## Recovery test: October 25, 2026

1. Restored the backup to **VM 331**, keeping production VM 330 running.
2. Renamed it `fastchannels-restore-test`.
3. Disabled automatic startup with `onboot: 0`.
4. Disconnected its virtual NIC using `link_down=1` to prevent an IP/hostname conflict.
5. Booted VM 331 through Proxmox and used the local console.
6. Confirmed its FastChannels web endpoint at `http://127.0.0.1:5523/admin/` responded **HTTP 200**.
7. Shut down VM 331, destroyed the temporary VM and its disk, and checked that production VM 330 was still running and the original backup remained available.

This establishes that the backup can be decompressed, restored, booted, and used to start the web interface. It does not establish comprehensive stream playback or end-to-end recording recovery.

## Remaining work

- Monitor filesystem growth and leave space for future container image updates.
- Recheck streams and EPG after provider-side changes.
- Periodically test recovery from a newer backup.
- Keep tokens, account credentials, and private provider URLs out of this public repository.
- Treat source-count and channel-number records as snapshots, not permanent guarantees.
