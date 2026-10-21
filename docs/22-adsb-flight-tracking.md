# ADS-B Flight Tracking Receiver

## Overview

One of the projects in my homelab is an ADS-B receiver that tracks aircraft flying nearby.

ADS-B stands for *Automatic Dependent Surveillance–Broadcast*. Aircraft equipped with ADS-B transmit information about themselves over radio, including their position, altitude, speed, and identification.

A receiver on the ground can pick up those broadcasts without needing to contact the aircraft or request the information.

The system uses a small USB software-defined radio and a computer to decode the signals. Software then displays the aircraft on a map and can share the information with flight-tracking networks.

I originally ran this on a Raspberry Pi 3 Model B.

In August 2026, I migrated the receiver to a Debian virtual machine in my Proxmox cluster.

The VM runs the receiver software, provides a local aircraft map, and shares data with FlightAware and Flightradar24.

---

## 1. How the System Works

The main components are:

- **RTL-SDR receiver:** Receives aircraft radio transmissions.
- **readsb:** Decodes ADS-B messages into aircraft data.
- **PiAware:** Sends decoded data to FlightAware.
- **FR24feed:** Sends data to Flightradar24.
- **tar1090:** Displays aircraft positions on a locally hosted map.

The signal path is:

```text
Aircraft ADS-B transmissions
             |
             v
       RTL-SDR receiver
             |
       USB passthrough
             |
             v
      Proxmox node pve03
             |
             v
      adsb01 Debian VM
             |
             v
           readsb
          /  |   \
         /   |    \
        v    v     v
   PiAware FR24  tar1090
      |      |      |
      v      v      v
 FlightAware FR24  Local map
```

The important design choice is that `readsb` controls the physical receiver.

The other programs use decoded data from `readsb` rather than attempting to open the same USB radio independently.

---

## 2. Virtual Machine Configuration

The receiver runs as `adsb01`, VMID 103, on `pve03`.

| Item | Configuration |
|---|---|
| Proxmox node | `pve03` |
| VMID | 103 |
| Hostname | `adsb01` |
| Recorded IP | `192.168.1.246` |
| Operating system | Debian 13 (Trixie) |
| Architecture | amd64 |
| vCPU | 2 |
| RAM | 2 GiB |
| Disk | 16 GiB |
| Networking | VirtIO on `vmbr0` |
| QEMU Guest Agent | Enabled |

The recorded address was provided by a fixed DHCP allocation.

The radio receiver is physically connected to `pve03`, so the VM depends on that host for its USB device.

---

## 3. RTL-SDR USB Receiver

The ADS-B receiver uses a Realtek RTL2832U device with an R820T tuner.

The recorded device details are:

| Field | Value |
|---|---|
| USB ID | `0bda:2832` |
| Tuner | Rafael Micro R820T |
| RTL-SDR serial | `00001000` |
| Proxmox configuration | `usb0: host=0bda:2832` |

On `pve03`, I can check whether Linux detects the USB receiver:

```bash
lsusb | grep 0bda:2832
```

I can also check whether it is assigned to VM 103:

```bash
qm config 103 | grep usb
```

Inside `adsb01`, I verify that the USB device is visible:

```bash
lsusb | grep 0bda:2832
```

If the receiver disappears after maintenance or a reboot, these checks help separate USB passthrough problems from issues with the decoding software.

---

## 4. readsb

`readsb` receives and decodes the aircraft transmissions.

The version recorded during the migration was:

```text
readsb 3.16.16
```

It provides decoded messages to the other services over local TCP ports.

| Port | Data format | Purpose |
|---|---|---|
| 30002 | Raw / AVR | FR24feed input |
| 30003 | SBS / BaseStation | Local output |
| 30004 | Beast | Input |
| 30005 | Beast | PiAware input |
| 30104 | Beast | MLAT return input |

To check the service:

```bash
systemctl status readsb --no-pager
```

### Checking Aircraft Reception

`readsb` maintains a JSON file containing recently received aircraft information.

On `adsb01`:

```bash
python3 - <<'PY'
import json

with open("/run/readsb/aircraft.json") as f:
    data = json.load(f)

print("Messages:", data.get("messages"))
print("Aircraft:", len(data.get("aircraft", [])))
PY
```

This is useful for confirming that the receiver is actually decoding aircraft messages.

A running `readsb` service alone doesn't guarantee that the antenna, USB receiver, or radio reception is working.

---

## 5. FlightAware and PiAware

PiAware forwards aircraft data to FlightAware.

During migration, I built PiAware for Debian 13 amd64 using the FlightAware builder.

The recorded version was:

```text
PiAware 11.1
```

PiAware receives Beast-format data from `readsb` on localhost.

The documented configuration is:

```text
receiver-type = other
receiver-host = 127.0.0.1
receiver-port = 30005
allow-mlat = yes
```

I retained the existing FlightAware feeder identity during migration rather than registering it as a completely new receiver.

### Verification

Useful commands include:

```bash
piaware-status
piaware-config -showall
journalctl -u piaware -n 50 --no-pager
```

I also check MLAT configuration:

```bash
piaware-config -show allow-mlat
```

### What MLAT Does

MLAT stands for *multilateration*.

It can help locate aircraft using the timing of radio signals received by multiple ground stations.

This is particularly useful for aircraft that transmit signals without including their own GPS-derived position.

Participation depends on the receiver network and the availability of suitable aircraft signals.

---

## 6. Flightradar24

FR24feed forwards decoded aircraft data to Flightradar24.

The version installed during migration was:

```text
FR24feed 1.0.57-1
```

The working configuration uses an AVR TCP feed from `readsb`.

The relevant section of `/etc/fr24feed.ini` is:

```ini
receiver="avr-tcp"
host="127.0.0.1:30002"
bs="no"
raw="no"
mlat="no"
mlat-without-gps="no"
```

The FR24 sharing key is stored locally and is not included in the repository.

### Checking the Service

```bash
systemctl status fr24feed --no-pager
journalctl -u fr24feed -n 40 --no-pager
```

The migration logs recorded a successful connection to the local receiver and an active upload connection.

The local FR24 status page was available at:

```text
http://192.168.1.246:8754/
```

### A Configuration Problem I Encountered

During the migration, I used the FR24 web settings interface to save receiver settings.

That replaced the working `avr-tcp` configuration with `dvbt`.

The change was inappropriate for this arrangement because `readsb`, not FR24feed, was supposed to control the physical receiver.

I restored the working configuration in `/etc/fr24feed.ini`.

Since then, I prefer to manage this part of the FR24 configuration directly from the shell.

This keeps the architecture clear: one program owns the USB radio, and the other services consume its decoded output.

---

## 7. Local Aircraft Map

I use `tar1090` to display nearby aircraft on a local map.

It consumes the decoded aircraft data provided by `readsb` and serves a browser-based interface.

The relevant services are:

```bash
systemctl status tar1090 --no-pager
systemctl status lighttpd --no-pager
```

The documented URL is:

```text
http://192.168.1.246/tar1090/
```

From inside the VM, I can check the HTTP response:

```bash
curl -I http://127.0.0.1/tar1090/
```

The map is useful for confirming aircraft reception visually.

It also makes the project more interesting than simply sending data to an external service.

---

## 8. Service Health Checks

The main services are:

- `readsb`
- `piaware`
- `fr24feed`
- `tar1090`
- `lighttpd`
- `qemu-guest-agent`

I can check them together:

```bash
systemctl is-active \
  readsb \
  piaware \
  fr24feed \
  tar1090 \
  lighttpd \
  qemu-guest-agent
```

The expected healthy state is `active` for each required service.

I also inspect the listeners:

```bash
sudo ss -lntp | grep -E ':80|:30002|:30003|:30005|:8754'
```

For complete validation, I want to see more than active processes.

I also check that `readsb` is receiving messages, the map works, and both external feeding services are connected.

---

## 9. Reboot Validation

After the migration, I performed a complete VM reboot and checked whether the receiver and services returned automatically.

The verification included:

```bash
systemctl is-active readsb piaware fr24feed tar1090 lighttpd qemu-guest-agent

lsusb | grep 0bda:2832

piaware-status

journalctl -u fr24feed -n 15 --no-pager
```

The RTL-SDR remained available through USB passthrough, and the core services restarted successfully.

This mattered because a receiver that works only until its next reboot isn't particularly useful as an unattended service.

---

## 10. Monitoring

The original monitoring plan included these Uptime Kuma checks:

| Monitor | Type | Target |
|---|---|---|
| `adsb01` host | Ping | `192.168.1.246` |
| `readsb` Beast feed | TCP | `192.168.1.246:30005` |
| `tar1090` map | HTTP | `http://192.168.1.246/tar1090/` |
| FR24feed status | HTTP | `http://192.168.1.246:8754/` |

These checks cover reachability and basic service availability.

Actual aircraft-message reception and successful external data feeding still require application-level checks.

---

## 11. Operational Dependencies

The physical receiver remains attached to `pve03`.

That creates a hardware dependency that isn't present for ordinary Proxmox virtual machines.

Moving `adsb01` to another node would require moving the USB receiver or redesigning how the radio data reaches the VM.

This is one reason I document USB passthrough separately from the rest of the VM configuration.

The former Raspberry Pi 3 receiver can be repurposed once the VM-based installation is operating reliably.

---

## Result

The ADS-B receiver was successfully migrated from a Raspberry Pi to a Debian VM running under Proxmox.

The migration preserved the existing aircraft-data feeds, added the receiver to the virtualization environment, and provided a locally hosted aircraft map.

I also tested USB passthrough, service startup, and operation after reboot.

The FR24 configuration issue was a useful reminder that several applications may participate in one data pipeline, but they shouldn't all attempt to control the same hardware.

This remains one of the more enjoyable services in the lab because it combines radio reception, Linux administration, virtualization, networking, and a practical way to visualize the data.

## Related Documentation

- [Proxmox Cluster Build](03-proxmox-cluster-build.md)
- [Asset Inventory](02-asset-inventory.md)
- [Homelab Operations Checklist](15-homelab-operations-checklist.md)
