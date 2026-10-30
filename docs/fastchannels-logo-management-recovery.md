# FastChannels Logo Management and Recovery

## Status — October 30, 2026

I repaired the logo references in **FC STe01** (FastChannels source ID `54`) on `fastchannels01 (192.168.1.228)`. The final checks showed:

| Metric | Verified result |
|---|---:|
| Source channels | 700 |
| Logos pinned to local HTTP files | 395 |
| Remaining `iptv4.net` logo URLs | 0 |
| Missing logo URLs | 0 |
| Full FastChannels `/m3u` entries across sources | 2,474 |

The final Canada One entry was present in the live M3U with its correct `tvg-logo` URL. Five earlier replacements were also explicitly verified in the live M3U, and the later replacements were confirmed in the application database. This is a **source 54** result, not a claim about every FastChannels source or downstream Channels DVR imports.

## Architecture

- FastChannels API/admin: `fastchannels01 (192.168.1.228:5523)`
- Logo HTTP server: `fastchannels01 (192.168.1.228:8766)`
- Working files: `/home/jatkins/channel-logos/`
- Logo PNGs: `/home/jatkins/channel-logos/logos/`
- Manager: `/home/jatkins/channel-logos/fc-logo-manager.py`
- SQLite database (inside Docker volume): `/var/lib/docker/volumes/fastchannels_fastchannels_data/_data/fastchannels.db`
- Downstream DVR: `docker (192.168.1.174:8089)`; this source is **not** currently a separately identified Channels DVR lineup.

The manager reads SQLite in read-only mode and sends changes through `PATCH http://127.0.0.1:5523/api/channels/{id}`. It refuses already pinned or changed channels, checks local PNG signatures and HTTP availability, and writes a rollback JSON snapshot **before** changing entries.

The `/var/lib/docker` directory is not traversable by the regular `jatkins` account. Use `sudo` for manager inventory and apply/rollback; do **not** relax Docker directory permissions.

## Read-only health checks

Run on **fastchannels01 (192.168.1.228)**:

```bash
cd /home/jatkins/channel-logos
sudo python3 fc-logo-manager.py inventory

curl -fsS --max-time 15 -o /dev/null \
  -w 'FastChannels M3U HTTP %{http_code}\n' \
  http://127.0.0.1:5523/m3u

curl -fsS --max-time 15 -o /dev/null \
  -w 'Canada One PNG HTTP %{http_code}\n' \
  http://192.168.1.228:8766/logos/2298-canada-one-review.png
```

Check `tvg-logo` inside the live M3U, not just the database. A directory URL such as `/logos/` may return HTTP 403 even when individual PNGs return HTTP 200. A full `/m3u` export includes multiple sources; it is not limited to FC STe01.

## Safe logo changes

The approved manifest format is:

```csv
channel_id,local_filename,approved
2298,2298-canada-one-review.png,yes
```

Use a **new** manifest for each approved batch. Never rerun a previously completed approved manifest unchanged: the manager intentionally refuses already pinned channels.

```bash
cd /home/jatkins/channel-logos
sudo python3 fc-logo-manager.py apply --manifest approved-BATCH.csv
# Review preflight output carefully before changing anything:
sudo python3 fc-logo-manager.py apply --manifest approved-BATCH.csv --execute
sudo python3 fc-logo-manager.py inventory
```

The above `approved-BATCH.csv` is a placeholder; use a real manifest for a future approved change. Do not paste `--execute` until the dry run is correct.

Rollback snapshots are named `rollback-managed-*.json`, and contain original channel states. To inspect a rollback candidate without modifying anything:

```bash
cd /home/jatkins/channel-logos
sudo python3 fc-logo-manager.py rollback --backup rollback-managed-TIMESTAMP.json
# Only after reviewing the dry run:
# sudo python3 fc-logo-manager.py rollback --backup rollback-managed-TIMESTAMP.json --execute
```

Rollback is selective: if a channel changed after the original operation, the manager skips it.

## Backup: preserve **both** the database and the logo directory

The 395 pinned assignments require both the FastChannels application state and the files served by port 8766. The existing daily Proxmox backup for **VM 330 on pve04** provides a VM-level recovery path, but a separate logical backup of the logo files and SQLite database makes point recovery easier. The VM restore tested on October 25 predates the October 30 logo cleanup; it does **not** prove the newest logo state is recoverable.

Before running a new logical backup, verify free space and choose a destination on storage separate from the FastChannels VM. Do not treat an archive stored solely on the VM as an off-host backup.

For a quick **local staging** copy of files and manifests:

```bash
cd /home/jatkins
stamp=$(date +%Y%m%d-%H%M%S)
mkdir -p "$HOME/logo-backup-staging"
tar -czf "$HOME/logo-backup-staging/channel-logos-$stamp.tar.gz" channel-logos
tar -tzf "$HOME/logo-backup-staging/channel-logos-$stamp.tar.gz" | head
```

This may include rollback metadata and source URLs; keep the archive private and do **not** commit it to a public repository. Copy it to an independently backed-up destination only after checking capacity and permissions. Verify the transferred archive using `sha256sum` on both sides.

For SQLite, prefer SQLite's consistent online-backup API rather than copying a live `.db` file with `cp`:

```bash
sudo python3 - <<'PY'
import sqlite3
from pathlib import Path
source = Path('/var/lib/docker/volumes/fastchannels_fastchannels_data/_data/fastchannels.db')
dest = Path('/home/jatkins/logo-backup-staging/fastchannels.sqlite3')
with sqlite3.connect(f'file:{source}?mode=ro', uri=True) as src:
    with sqlite3.connect(dest) as dst:
        src.backup(dst)
with sqlite3.connect(dest) as check:
    print('Integrity:', check.execute('PRAGMA integrity_check').fetchone()[0])
PY
```

**Important:** that SQLite snippet needs the staging directory created first (see the tar command), and produces a root-owned local staging file. Inspect disk space, secure access to the staging directory, and arrange off-host transfer before treating it as protected. Run a new VM 330 backup after these changes and test restore in an isolated VM before declaring recovery proven.

## Recovery order (procedure, not yet exercised for the October 30 state)

1. On an isolated recovery system, verify the FastChannels VM/container and database version are compatible.
2. Restore the FastChannels application state using the appropriate VM or application recovery method. Never overwrite a live SQLite database while FastChannels is using it.
3. Restore the `channel-logos` directory, keeping its `logos/` filenames and directory structure intact.
4. Restore the HTTP service on port **8766** using its actual verified service configuration; the service unit or launch method still needs to be captured and backed up.
5. Check reachability of several exact PNG URLs from `fastchannels01` and `docker`.
6. Check `fc-logo-manager.py inventory`, then inspect the live `/m3u` for the expected `tvg-logo` URLs.
7. Only after a successful isolated test, plan any production recovery or cutover. Avoid starting a restored VM on the same network with `192.168.1.228` while production is online.

**Not yet verified:** independent off-host copy of the October 30 logical backup; startup method for port 8766; and an isolated restoration of the post-cleanup 395-logo state.

## References

- [Channels DVR and FastChannels Media Services](27-channels-dvr-fastchannels.md)
- [FastChannels EPG Refresh and Monitoring](fastchannels-epg-refresh-and-monitoring.md)
- [Backup Restore Validation](21-proxmox-backup-restore-validation.md)
