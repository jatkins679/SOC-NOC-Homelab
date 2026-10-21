# Tailscale Remote Access and Troubleshooting

## Overview

I use Tailscale to access my homelab when I'm away from home.

The main objective is to reach internal systems without exposing Proxmox, SSH, RDP, or other management services directly to the public Internet.

I originally configured a Raspberry Pi as a dedicated Tailscale subnet router for my home network.

That arrangement generally worked, but traveling in Europe in October 2026 exposed a problem with the addressing scheme I'd been using.

The remote network and my home network both used `192.168.1.0/24`.

As a result, my Mac attempted to reach home LAN addresses through its local Wi-Fi connection rather than through Tailscale.

The investigation also involved a separate direct Tailscale connectivity problem, which I eventually worked around using an SSH tunnel.

This page documents the configuration, the troubleshooting steps, and what I verified.

---

## 1. Remote Access Architecture

The main remote-access system is `tailscale01`, a Raspberry Pi running Debian 13.

| Item | Recorded configuration |
|---|---|
| Hostname | `tailscale01` |
| Operating system | Debian 13, arm64 |
| LAN address | `192.168.1.185` |
| Tailscale address | `100.90.238.71` |
| Advertised subnet | `192.168.1.0/24` |
| Function | Dedicated subnet router |

These addresses were recorded during the October 2026 investigation. They should be checked against Tailscale and the live host before being treated as current.

The original traffic path is:

```text
Remote laptop
     |
     v
Tailscale encrypted connection
     |
     v
tailscale01
192.168.1.185
     |
     v
Home LAN
192.168.1.0/24
     |
     +-- Proxmox nodes
     +-- Storage server
     +-- DNS and management services
```

The Raspberry Pi provides remote access independently of the Proxmox cluster.

That means I can still reach the original LAN even if an individual hypervisor is down.

It does not automatically provide access to every segmented VLAN. Inter-VLAN access still depends on OPNsense and its firewall rules.

---

## 2. Subnet Routing

Tailscale connects devices using their assigned `100.x.x.x` addresses.

Subnet routing extends that access to devices that aren't running Tailscale themselves.

For example, a remote laptop can communicate with `pve02` at `192.168.1.11` through `tailscale01`, provided the remote client accepts the route and local routing doesn't override it.

On the subnet router, IPv4 forwarding is enabled.

The route `192.168.1.0/24` was advertised and approved in Tailscale.

Useful Linux checks include:

```bash
tailscale status
tailscale ip -4
ip -4 route
sysctl net.ipv4.ip_forward
```

On a Mac using the Tailscale application, the command-line executable may not be in the shell's default `PATH`.

I can call it directly:

```bash
/Applications/Tailscale.app/Contents/MacOS/Tailscale status
```

The original subnet-router configuration was successfully used to reach systems on the home LAN.

---

## 3. The Problem While Traveling

In October 2026, I was traveling in Europe and wanted to remotely access `storage01`, my Windows storage server.

The server had two relevant addresses:

| Network | Address |
|---|---|
| Home LAN | `192.168.1.208` |
| Tailscale | `100.121.10.38` |

Initially, neither address worked reliably from my traveling Mac.

Remote Desktop failed, and Tailscale initially reported the Windows server as offline.

I wanted to establish whether the problem was with the server, RDP, Tailscale, or routing.

---

## 4. Confirming the Server Was Running

My first useful step was to connect to `tailscale01`, which remained accessible through Tailscale.

From there, I tested `storage01` over the home LAN.

```bash
ping -c 4 192.168.1.208
```

All four pings succeeded.

I then checked RDP connectivity:

```bash
nc -vz -w 5 192.168.1.208 3389
```

TCP port 3389 accepted the connection.

These tests established that the Windows server was running and that RDP was reachable from within the home network.

That narrowed the problem considerably.

It also meant there was no immediate reason to modify Windows firewall rules or restart the storage server.

---

## 5. Finding the Overlapping Subnets

On my Mac, I checked the route to the home LAN address:

```bash
route -n get 192.168.1.208
```

I also checked the Wi-Fi address and routing table:

```bash
ipconfig getifaddr en0
netstat -rn -f inet | grep 192.168.1
```

The traveling Mac had received:

```text
192.168.1.179
```

The remote Wi-Fi network was using the same `192.168.1.0/24` subnet as my home network.

The route to `192.168.1.208` selected the Mac's local Wi-Fi interface, `en0`, rather than its Tailscale interface.

The destination appeared to be on the local network.

That explained why the home LAN address didn't work.

The connection attempts weren't reaching the subnet router.

### Why This Happened

Both networks were using:

```text
192.168.1.0/24
```

That is a common default address range on consumer routers.

The operating system preferred its directly connected local subnet for the destination.

Even though Tailscale advertised the home subnet, the overlapping local route took precedence.

This was a routing problem, not a failure of the Windows server or RDP service.

---

## 6. Using an SSH Tunnel

Because `tailscale01` could still reach the Windows server, I used it as an SSH jump host.

From a terminal on the Mac:

```bash
ssh -N -L 13389:192.168.1.208:3389 jatkins@100.90.238.71
```

The command creates a local TCP listener on port 13389 and forwards connections through the SSH session to port 3389 on `storage01`.

The path is:

```text
Remote Desktop on Mac
127.0.0.1:13389
       |
       v
Local SSH tunnel
       |
       v
tailscale01
100.90.238.71
       |
       v
storage01
192.168.1.208:3389
```

I then connected the Remote Desktop client to:

```text
127.0.0.1:13389
```

After correcting the local tunnel setup, the RDP session worked.

The first attempt produced a localhost connection-refused error. I didn't preserve enough evidence to determine the exact cause, so I won't assign one.

### Checking the Tunnel

For future troubleshooting, useful Mac commands are:

```bash
lsof -nP -iTCP:13389 -sTCP:LISTEN
```

and:

```bash
ssh -v -o ExitOnForwardFailure=yes -N \
  -L 13389:192.168.1.208:3389 \
  jatkins@100.90.238.71
```

The first command confirms whether something is listening on the local port.

The second provides SSH diagnostic output and detects an unsuccessful local port-forward setup.

The SSH session must remain running while the tunnel is in use.

---

## 7. Restoring Direct Tailscale Access

Once I had access to `storage01`, I checked its Tailscale service.

From PowerShell on the Windows server:

```powershell
Get-Service Tailscale
tailscale status
```

The service was running.

I also tested Tailscale connectivity back to the traveling Mac:

```powershell
tailscale ping 100.101.215.58
```

That succeeded.

From the Mac, I then tested the reverse direction:

```bash
/Applications/Tailscale.app/Contents/MacOS/Tailscale ping 100.121.10.38
```

Tailscale initially used a DERP relay in Madrid, then established a direct peer connection.

Next, I tested RDP:

```bash
nc -vz -G 5 100.121.10.38 3389
```

TCP 3389 was reachable.

I successfully established a direct Remote Desktop connection to:

```text
100.121.10.38
```

At that point, the SSH tunnel was no longer necessary.

### What Remained Unexplained

The earlier Tailscale offline indication cleared during the investigation.

I confirmed that the connection eventually worked, but I did not establish exactly why the Windows server had initially appeared offline to the Mac.

That uncertainty is separate from the overlapping-subnet problem, which was directly demonstrated by the Mac's routing table.

---

## 8. A Separate Surfshark Problem

During the same trip, I encountered another connectivity problem on a Mac.

The Mac connected to Wi-Fi and could ping its local gateway, but it couldn't reach Internet destinations by hostname or IP address.

The symptom occurred on more than one Wi-Fi network.

The cause was Surfshark VPN, which was still connected.

Disconnecting Surfshark restored Internet access.

This incident wasn't caused by the Tailscale subnet router or by the overlapping home and remote networks.

It was a separate VPN connectivity problem.

### Diagnostic Commands

Useful read-only commands on macOS include:

```bash
route -n get default
netstat -rn -f inet
scutil --nwi
ifconfig | grep -A 4 '^utun'
```

These help reveal routing and tunnel-interface state.

If the gateway responds but external IP addresses don't, I now check active VPN connections before changing DNS or Wi-Fi settings.

A successful local gateway ping doesn't establish that the machine has a working route to the Internet.

---

## 9. Tailscale and Commercial VPNs

I use Tailscale and Surfshark for different purposes.

Tailscale provides a private connection between my devices and my homelab.

Surfshark is a commercial VPN that primarily changes or protects outbound Internet connectivity.

Both can create tunnel interfaces and affect routing, but they aren't interchangeable.

When troubleshooting a Mac running more than one VPN application, I need to establish which tunnels are actually active and which routes they control.

I shouldn't assume that a VPN is disconnected merely because I don't see an application window open.

---

## 10. Remote Access Troubleshooting Checklist

When I can't reach a home system remotely, I use the following sequence.

1. Confirm that Tailscale is connected on the remote client.
2. Check whether the destination has a direct Tailscale address.
3. Test the peer using `tailscale ping`.
4. Test the required TCP port.
5. Inspect the remote client's routing table.
6. Check whether the remote network overlaps the home subnet.
7. Test the destination from a known-good system inside the home network.
8. Verify the actual application.
9. Use a temporary SSH tunnel if the destination is reachable through an intermediary.
10. Record which tests succeeded and which failed.

For systems reachable only through the subnet router, I also confirm the advertised route and whether the client is accepting it.

For hosts on VLANs behind OPNsense, I check the firewall and return-routing requirements separately.

The key is to establish where communication stops rather than changing settings on several systems at once.

---

## 11. Planned Home LAN Renumbering

The experience highlighted a limitation in my original home network design.

Using `192.168.1.0/24` makes subnet conflicts relatively likely when connecting from other homes, hotels, or travel networks.

I'm considering changing the original default LAN to:

```text
192.168.11.0/24
```

That change has not yet been implemented.

The existing segmented networks include:

| VLAN | Purpose |
|---|---|
| 10 | Management |
| 20 | Servers |
| 30 | Users |
| 40 | SOC/Security |
| 50 | Attack/Test |
| 60 | Guest |

Moving the remaining flat-LAN devices or changing the default subnet requires careful planning.

The dependencies include the AT&T gateway, Proxmox management addresses, switch management, Pi-hole, OPNsense, DHCP, static reservations, DNS, monitoring systems, Tailscale routes, and firewall rules.

I intend to make the changes incrementally and preserve a working management and rollback path.

Changing to a less common subnet reduces the likelihood of overlap but can't eliminate it completely.

[See the VLAN Migration Dependency Checklist](24-vlan-migration-dependency-checklist.md).

---

## Result

The Tailscale subnet router gives me an independent way to reach the home LAN from outside the network.

The October 2026 trip exposed two separate issues: a confirmed subnet overlap that prevented direct access to `192.168.1.x` destinations, and an initially unavailable direct Tailscale connection to the Windows storage server.

I verified that the server and RDP service were running, established access through an SSH tunnel, and later confirmed direct RDP using the server's Tailscale IP address.

The separate Surfshark incident also reinforced the need to check active VPN routes when diagnosing Internet connectivity.

These experiences are influencing the planned redesign of the default LAN addressing scheme and the way I approach remote-access troubleshooting.

## Related Documentation

- [Managed Switching and VLANs](04-network-vlans.md)
- [OPNsense Routing and Firewalling](05-opnsense.md)
- [NOC Troubleshooting Runbook](13-noc-troubleshooting-runbook.md)
- [VLAN Migration Dependency Checklist](24-vlan-migration-dependency-checklist.md)
- [Asset Inventory](02-asset-inventory.md)
