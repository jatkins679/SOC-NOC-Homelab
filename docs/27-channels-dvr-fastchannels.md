# Channels DVR and FastChannels Media Services

## Overview

I wanted to bring the free live-TV sources I've been experimenting with into one guide on Apple TV. I already had Dispatcharr and IPTVX running, but I also wanted to try Channels DVR and see how well it worked with the different free providers.

I installed Channels DVR on my existing `docker` VM and set up FastChannels on its own Proxmox VM. Keeping FastChannels separate was deliberate: an earlier attempt on the Docker VM was followed by serious memory pressure and a lockup. I didn't establish that FastChannels alone caused the problem, but I didn't want another source scan affecting the rest of my services.

By October 25, 2026, the feeds were working, Uptime Kuma was monitoring FastChannels, and I'd completed a backup and an isolated restore test. The restore confirmed that Ubuntu and the FastChannels web interface would start. I haven't tested every individual channel from a restored VM.

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

All of these addresses are on my internal network. I'm not publishing the services directly to the Internet. The earlier IPTVX recording setup is documented separately in [IPTV DVR and Recording Storage Monitoring](26-iptv-dvr-recording-monitoring.md).

## System inventory

| Component | Location | Configuration |
|---|---|---|
| Channels DVR | `docker`, VM 210 on `pve02` | `192.168.1.174:8089` |
| Dispatcharr | Docker on `docker` | `192.168.1.174:9191` |
| Samsung TV Plus connector | Docker on `docker` | `192.168.1.174:8182` |
| FastChannels | `fastchannels01`, VM 330 on `pve04` | `192.168.1.228:5523` |
| Uptime Kuma | Docker on `docker` | `192.168.1.174:3001` |
| Shared backup storage | `storage01` | `192.168.1.208`, CIFS `t-20-backup` |

For the dedicated VM, I allocated two vCPUs, 4 GiB RAM, and a 24 GiB virtual disk and installed Ubuntu Server 24.04. I keep its Docker Compose files in `/home/jatkins/fastchannels`. The container has a 2 GiB memory limit, a combined 3 GiB memory-and-swap limit, and two persistent named volumes for application data and Android Debug Bridge data.

After checking that the container was running normally, I changed its restart policy from `no` to `unless-stopped`. I also enabled VM 330's automatic startup in Proxmox with `onboot: 1` and `startup: order=3,up=30`. The 30 seconds is the delay **after** this VM starts, before Proxmox starts the next VM. It isn't a delay before FastChannels boots.

## Source feeds and numbering

I added providers one at a time: scrape, audit the channels, approve the ones that work, and create a separate feed for each provider. I didn't import the Default feed because that would make it too easy to bring in the same channels twice. Each dedicated feed uses one source and the mixed playlist/guide option.

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

The table records the starting numbers I used or planned during setup. Before adding another feed, I check the current assignments to avoid conflicts. I confirmed that the integrated sources were working in Channels DVR, including Local Now's playback and guide data. That doesn't mean every single stream will always work. Some PBS channels are DRM-protected, and I left blocked or dead Free Networks streams out rather than trying to work around them.

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

I use the FastChannels admin page to check source status, audits, channel approvals, and feed settings. Once a feed is ready, I use its **Add to Channels DVR** action. The Apple TV then uses Channels DVR rather than connecting to each provider separately.

### Resource investigation

At one point I saw Docker report about 101% CPU usage. I checked the running processes and logs instead of changing the limits immediately. Zeam was scraping and auditing hundreds of channels. Later, at idle, FastChannels was using about 533 MiB of RAM and 0.45% CPU.

The root filesystem was already 63% full, with about 7.8 GiB free. Docker reported a 7.6 GB FastChannels image and around 634 MB of persistent data. Those numbers use different accounting methods, so I didn't try to add them together to explain the filesystem total.

I left the installation alone instead of running `docker system prune`. I'll keep an eye on free space, especially before updating the image.

### Uptime Kuma

Uptime Kuma on `docker (192.168.1.174:3001)` has an HTTP monitor named `fastchannels01 - FastChannels` for:

```text
http://192.168.1.228:5523/admin/
```

I set the monitor to check every 60 seconds, with three retries and a 10-second timeout. It turned green after I saved it. That tells me the web interface is answering, but not whether individual streams or guide updates are working.

## Backup configuration

I checked the backup jobs before adding anything new. `pve04` already had a daily **all-VM** backup, so VM 330 was covered without creating another job:

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

The first backup was about 6.8 GB. I ran `zstd -t` against it, and the compressed archive passed. That's worth checking, but it isn't the same as actually restoring the VM. Also, because the backup runs in snapshot mode, files being written during the backup might not be fully application-consistent.

## Recovery test: October 25, 2026

A backup file doesn't do me much good if I can't restore it. I restored this one to **VM 331**, leaving production VM 330 running. The restored VM initially had the same hostname, network settings, and automatic-start configuration as the original, so I changed those before booting it.

I renamed the test machine `fastchannels-restore-test`, turned off automatic startup (`onboot: 0`), and disconnected its virtual network adapter (`link_down=1`). I wanted to avoid having two machines trying to use `192.168.1.228`.

I booted VM 331 and logged in through the Proxmox console. The local FastChannels page at `http://127.0.0.1:5523/admin/` returned **HTTP 200**. That was enough to verify the basic restore and application startup. It wasn't a full test of every TV stream or the DVR recording path.

Once I'd finished, I shut down and deleted VM 331 and its disk. I checked that VM 330 was still running and that the backup file was still on `t-20-backup`.

## OliveTin-for-Channels maintenance note — October 26, 2026

I added OliveTin-for-Channels on `docker (192.168.1.174)` to manage Channels DVR at `192.168.1.174:8089`. The authenticated web dashboard works under my `jatkins` account. Its post-install check reached Channels DVR with HTTP 200 and showed the DVR data and log mounts. The channel-inventory action successfully found 14 DVR sources and wrote a CSV file.

I also enabled **Channel Lineup Change Notifications** at a 60-minute interval, with local logs only (no email or SMS). Its first run failed because a source named `FastChannels PBS / Public Television` contained a slash. The Python script used a source name directly to construct a log filename without replacing path separators. The verified correction in `/home/jatkins/olivetin/config/channels_dvr_monitor_channels.py` is:

```python
return os.path.join(
    Data_Subdirectory_Path,
    name.replace(' ', '_').replace('/', '_').replace(chr(92), '_')
)
```

The original script was backed up as `channels_dvr_monitor_channels.py.backup-20261027-010025`. After the patch, the monitor generated `FastChannels_PBS___Public_Television.log` for the 29-channel source and continued running.

### Update protection and maintenance

I changed `/home/jatkins/olivetin/compose.yaml` to:

```yaml
UPDATE_YAMLS: "false"
UPDATE_SCRIPTS: "false"
```

I backed up the Compose file, validated it with `docker compose config --quiet`, then recreated only the OliveTin service. `docker inspect` confirmed both settings were `false`. The corrected script remained in place, and the monitor was running again after the recreation.

**This is a temporary local patch.** Until the upstream monitor script fixes filename sanitization, leave automatic script replacement disabled. Periodically review upstream releases manually, compare the affected function, back up the working version, and only re-enable `UPDATE_SCRIPTS` after testing the replacement. The change does not freeze the Docker image itself or prevent someone from manually overwriting the script.

Useful read-only checks on `docker (192.168.1.174)`:

```bash
docker inspect olivetin --format '{{range .Config.Env}}{{println .}}{{end}}' | grep '^UPDATE_'
docker exec olivetin sh -c "ps aux | grep '[c]hannels_dvr_monitor_channels.py' || true"
docker exec olivetin tail -15 /config/192.168.1.174-8089_data/last_activity.log
grep -n -A 12 'def create_local_file_name' /home/jatkins/olivetin/config/channels_dvr_monitor_channels.py
```

The Docker API socket remains deliberately unmounted from OliveTin. Docker-dependent actions produce socket errors but Channels DVR operations work. Two startup actions also log ACL denials under the separate `startup` identity; I haven't relaxed the access controls to address those warnings.

The complete investigation and verification are recorded in [Ticket 17 — OliveTin Channel Lineup Monitor](14-soc-noc-ticket-examples.md). No account credentials, access tokens, or subscription stream URLs are stored in this documentation.

---

## Remaining work

- Keep an eye on disk space before future Docker image updates.
- Recheck channel streams and guide data when a provider changes something.
- Restore another backup periodically instead of assuming the daily job is enough.
- Keep tokens, passwords, and private provider URLs out of this public repository.

Channel counts and numbering in this page are a record of the setup at this point in time, not a promise that the provider catalogs will stay the same.
