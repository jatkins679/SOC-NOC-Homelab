# Tailscale Remote Access

## Purpose

This document records the implemented remote-access path for the homelab using a
dedicated Tailscale subnet router.

The objective is to provide authenticated remote access to systems on the existing
`192.168.1.0/24` LAN without exposing management services directly to the public
Internet.

## Implemented Host

| Item | Value |
|---|---|
| Hostname | `tailscale01` |
| Platform | Raspberry Pi |
| OS | Debian GNU/Linux 13 (trixie), arm64 |
| LAN address | `192.168.1.236` |
| Tailscale address | `100.90.238.71` |
| Advertised route | `192.168.1.0/24` |
| Role | Dedicated Tailscale subnet router |
| Status | **Operational** |

IPv4 forwarding was enabled as part of the subnet-router configuration.

## Traffic Path

```text
Remote Tailscale Client
        |
        | encrypted Tailscale tunnel
        v
tailscale01
192.168.1.236
        |
        | advertised route 192.168.1.0/24
        v
Home / Homelab LAN
```

This design keeps Tailscale's role separate from the AT&T gateway and from future
OPNsense/VLAN work.

## Design Rationale

A dedicated Raspberry Pi was selected so remote access does not depend on any one
Proxmox node remaining online.

This provides:

- an independent path to the flat management LAN;
- access to SSH, web-management, and other internal services through Tailscale;
- a clean separation between remote-access networking and experimental VM workloads;
- a reusable platform for future remote-management testing.

## Tailscale vs Commercial VPN

Tailscale is used here for **private remote access into the homelab**.

A commercial VPN service such as Surfshark serves a different purpose: it primarily
changes or protects outbound Internet traffic. It is not a substitute for the
Tailscale subnet-router role documented here.

## Validation

The implementation was considered operational after:

- `tailscale01` joined the tailnet;
- IPv4 forwarding was enabled;
- route `192.168.1.0/24` was advertised and approved;
- remote Tailscale clients could reach LAN systems through the subnet router.

## Security Notes

- Do not expose Tailscale authentication keys, reusable auth keys, or node secrets.
- Keep remote-access authorization in the Tailscale control plane.
- Continue to treat the local LAN as trusted only to the extent justified by the
  current flat-network design.
- Re-evaluate advertised routes and ACLs when OPNsense/VLAN segmentation is deployed.

## Skills Demonstrated

- VPN / overlay networking
- Linux IP forwarding
- subnet routing
- remote administration
- route advertisement
- secure management-plane design
- troubleshooting and validation
