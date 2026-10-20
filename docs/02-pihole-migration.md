# Pi-hole Migration to a Dedicated Raspberry Pi

## Overview

One of the first changes I made while rebuilding my homelab was to move Pi-hole DNS off Proxmox and onto a dedicated Raspberry Pi.

Pi-hole had been running in an LXC container. It worked, but it also meant that shutting down the Proxmox host could interrupt DNS service for the rest of the house.

Since I was planning to rebuild the Proxmox environment, that was a dependency I wanted to eliminate.

I already had an older Raspberry Pi available, so I decided to put it to use as a dedicated DNS server.

The migration is complete. Pi-hole now runs on `dns01` at `192.168.1.20`, independently of the Proxmox cluster.

---

## 1. Why Move DNS?

The original arrangement was straightforward:

```text
Home network clients
         |
    Pi-hole LXC
         |
    Proxmox host
         |
     Mini PC
```

The problem was that DNS availability depended on the Proxmox host.

Restarting the hypervisor, modifying network bridges, or rebuilding the system could also interrupt DNS for other devices on the network.

I wanted to separate those two functions.

The new arrangement is:

```text
Home network clients
         |
    dns01 — 192.168.1.20
         |
    Raspberry Pi
         |
      Pi-hole
```

The Raspberry Pi can continue providing DNS while the Proxmox cluster is undergoing maintenance.

This doesn't eliminate every possible DNS failure, but it removes a dependency that wasn't necessary.

---

## 2. Preparing the Raspberry Pi

The replacement system was an older Raspberry Pi Model B+ running Raspberry Pi OS.

| Setting | Value |
|---|---|
| Hostname | `dns01` |
| IP address | `192.168.1.20` |
| Service | Pi-hole DNS |
| Platform | Raspberry Pi Model B+ |
| Operating system | Raspberry Pi OS |

### Change the hostname

I renamed the Raspberry Pi to match its infrastructure role.

```bash
sudo hostnamectl set-hostname dns01
```

I then checked the local hosts file:

```bash
cat /etc/hosts
```

The old hostname, `blue-rp`, still appeared in the file, so I replaced it:

```bash
sudo sed -i 's/blue-rp/dns01/g' /etc/hosts
```

The `sed` command edits the file in place, replacing references to the old hostname with `dns01`.

After making the changes, I restarted the Raspberry Pi:

```bash
sudo reboot
```

This gave the DNS server a hostname consistent with the naming convention I was beginning to use throughout the lab.

### Update the operating system

Before installing Pi-hole, I updated Raspberry Pi OS:

```bash
sudo apt update
sudo apt full-upgrade -y
```

The first command refreshes package information. The second installs available upgrades and allows APT to resolve dependency changes.

---

## 3. Installing Pi-hole

I installed Pi-hole using the project's installation script:

```bash
curl -sSL https://install.pi-hole.net | bash
```

The command retrieves the installation script and passes it directly to Bash.

The interactive installer was then used to configure Pi-hole as the DNS service for the Raspberry Pi.

Once installation was complete, I checked its status:

```bash
pihole status
```

The output confirmed that the Pi-hole FTL service was running and listening for DNS queries on port 53.

It reported DNS listening over both TCP and UDP for IPv4 and IPv6, with blocking enabled.

This was the first confirmation that the new server was ready to provide DNS service.

---

## 4. Moving DNS to the Raspberry Pi

I wanted to retain the existing DNS service address:

`192.168.1.20`

That address had already been used by the original Pi-hole container. Keeping it meant I wouldn't need to update every client or network setting that referred to the DNS server.

The migration sequence was:

1. Prepare the Raspberry Pi and install Pi-hole.
2. Check that the new DNS service is functioning.
3. Configure the Raspberry Pi to use `192.168.1.20`.
4. Move DNS traffic to the new physical system.
5. Verify client DNS resolution and filtering.
6. Remove the old Pi-hole LXC from DNS service.

The old and new Pi-hole instances were not intended to use the same address simultaneously.

### Before

```text
192.168.1.20
     |
  Pi-hole LXC
     |
   Proxmox
```

### After

```text
192.168.1.20
     |
  Pi-hole
     |
 Raspberry Pi
```

Once the physical Pi-hole server was operating successfully at the established address, the old LXC was retired from DNS service.

---

## 5. Validating the Migration

After the cutover, I checked that the Raspberry Pi was reachable at `192.168.1.20` and that Pi-hole was running.

The validation covered several areas.

### Server

- Correct hostname: `dns01`
- Correct IP address: `192.168.1.20`
- Pi-hole FTL running
- DNS listening on port 53
- Filtering enabled

### Client DNS

I checked that clients could resolve public hostnames, access Internet services, and generate queries visible through Pi-hole.

This was important because a running DNS service isn't necessarily proof that clients are using it.

### Proxmox Independence

The principal architectural objective was to prevent Proxmox maintenance from taking DNS offline.

Moving Pi-hole onto dedicated hardware achieved that separation.

The original notes confirm successful cutover and continued DNS operation, although they don't include a complete terminal transcript of a Proxmox maintenance test.

---

## 6. Troubleshooting DNS

The migration also gave me a useful way to think about diagnosing DNS problems.

When a client can't resolve a hostname, I want to work through the relevant dependencies instead of changing several settings at once.

```text
Client DNS configuration
          |
Network connectivity
          |
dns01 — 192.168.1.20
          |
Pi-hole FTL
          |
Upstream DNS
```

For example, on a Linux client I can check its configured DNS resolvers and test the Pi-hole server directly:

```bash
resolvectl status
dig @192.168.1.20 example.com
```

On `dns01`, useful checks include:

```bash
ip -br addr
ip route
pihole status
```

These separate an addressing or connectivity issue from a DNS service problem.

The `dig` and `resolvectl` commands are diagnostic examples included for future use, not claims that they were captured in the original migration transcript.

---

## 7. Commands Used

| Command | Purpose |
|---|---|
| `sudo hostnamectl set-hostname dns01` | Rename the host |
| `cat /etc/hosts` | Inspect local hostname mappings |
| `sudo sed -i 's/blue-rp/dns01/g' /etc/hosts` | Replace the old hostname |
| `sudo reboot` | Restart the Raspberry Pi |
| `sudo apt update` | Refresh package information |
| `sudo apt full-upgrade -y` | Upgrade installed packages |
| `curl -sSL https://install.pi-hole.net \| bash` | Run the Pi-hole installer |
| `pihole status` | Check DNS and filtering status |

---

## 8. Result

Pi-hole now runs on a dedicated Raspberry Pi at:

```text
dns01
192.168.1.20
```

The original LXC no longer provides DNS service.

Keeping the existing IP address made the migration simpler, and moving the service onto separate hardware means I can restart or rebuild Proxmox without automatically taking down the network's DNS server.

It was a relatively small infrastructure change, but it removed an unnecessary dependency before I started the larger Proxmox rebuild.

---

## Related Documentation

- [Proxmox Backup and Disaster-Recovery Preparation](01-proxmox-backup.md)
- [Proxmox Cluster Build](03-proxmox-cluster-build.md)
- [Asset Inventory](02-asset-inventory.md)
