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
| LAN address | `192.168.1.185` |
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
192.168.1.185
        |
        | advertised route 192.168.1.0/24
        v
Home / Homelab LAN
```

This design keeps Tailscale's role separate from the AT&T gateway and from the OPNsense/VLAN routing policy used inside the segmented lab.

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
- Continue to treat the flat `192.168.1.0/24` LAN as trusted only to the extent justified by the current design.
- The advertised Tailscale route currently covers `192.168.1.0/24`; access to routed lab VLANs is governed separately by OPNsense policy and should not be assumed from the subnet-router advertisement alone.

## Skills Demonstrated

- VPN / overlay networking
- Linux IP forwarding
- subnet routing
- remote administration
- route advertisement
- secure management-plane design
- troubleshooting and validation


## Troubleshooting Case: Remote RDP Failure While Traveling (October 12, 2026)

While traveling in Spain, I couldn't reach \`storage01\` using either its home LAN address (\`192.168.1.208\`) or its Tailscale address (\`100.121.10.38\`). Remote Desktop failed, and initial ping attempts also failed. The Tailscale client on my Mac initially described the Windows server as offline.

Rather than start changing the Windows server or firewall rules, I worked through the connection from the inside out.

### What I checked

1. **Checked the Mac's Tailscale status.** On the Mac, the \`tailscale\` command wasn't in my shell's PATH, but the application binary worked:
   \`\`\`bash
   /Applications/Tailscale.app/Contents/MacOS/Tailscale status
   \`\`\`
   At first, \`storage01\` appeared as \`idle; offline\`, with transmitted bytes but no received bytes.

2. **Checked the server from the home network.** I SSHed to the subnet router, \`tailscale01\` (\`100.90.238.71\`), and ran:
   \`\`\`bash
   ping -c 4 192.168.1.208
   nc -vz -w 5 192.168.1.208 3389
   \`\`\`
   All four pings succeeded, and TCP port 3389 accepted a connection. That established that the server was online and the RDP port was reachable from within the home LAN.

3. **Looked at routing on the traveling Mac.**
   \`\`\`bash
   route -n get 192.168.1.208
   ipconfig getifaddr en0
   netstat -rn -f inet | grep 192.168.1
   \`\`\`
   My Mac had received \`192.168.1.179\` from the network in Spain. Its route to \`192.168.1.208\` selected \`en0\` (local Wi-Fi), not the Tailscale \`utun12\` interface. Both networks were using \`192.168.1.0/24\`. That was the confirmed reason the home LAN address wouldn't work from this location: traffic was going to the local network instead.

### Workaround: SSH tunnel to the RDP server

Because \`tailscale01\` could reach \`storage01\`, I used the subnet router as a jump host. From a **terminal on the Mac**, not from a shell on the Linux router:

\`\`\`bash
ssh -N -L 13389:192.168.1.208:3389 jatkins@100.90.238.71
\`\`\`

Then I connected the Remote Desktop client to \`127.0.0.1:13389\`. This provided access without depending on the conflicting \`192.168.1.x\` route from the Mac. I initially received a localhost connection-refused message; after resolving the local SSH tunnel setup, RDP through the tunnel succeeded. The exact cause of that first refusal wasn't recorded.

For future tunnel diagnostics on macOS:

\`\`\`bash
lsof -nP -iTCP:13389 -sTCP:LISTEN
ssh -v -o ExitOnForwardFailure=yes -N \
  -L 13389:192.168.1.208:3389 \
  jatkins@100.90.238.71
\`\`\`

The tunnel terminal needs to remain open while it's in use.

### Restoring direct Tailscale access

Once I was signed in to \`storage01\`, the Windows Tailscale service was running. In PowerShell:

\`\`\`powershell
Get-Service Tailscale
tailscale status
tailscale ping 100.101.215.58
\`\`\`

The Windows server could ping \`lisas-macbook-air\` (\`100.101.215.58\`) over Tailscale. From the Mac, the reverse test also succeeded:

\`\`\`bash
/Applications/Tailscale.app/Contents/MacOS/Tailscale ping 100.121.10.38
nc -vz -G 5 100.121.10.38 3389
\`\`\`

Tailscale first responded through the Madrid DERP relay and then over a direct peer connection. TCP port 3389 was reachable, and I successfully established **direct RDP to \`100.121.10.38\`**. At that point, the SSH tunnel was no longer needed.

The original offline indication cleared during testing, but I did not establish exactly why it appeared. It should not be confused with the separate, confirmed LAN subnet-overlap problem.

### Practical takeaways

- A host can be running and accepting RDP locally even when it's unreachable from a traveling laptop.
- Check the routing table before assuming Windows, RDP, or Tailscale itself is broken.
- When two locations use the same LAN subnet, the operating system may send traffic to local Wi-Fi instead of the remote subnet router.
- A Tailscale node's \`100.x.x.x\` address avoids the overlapping \`192.168.1.x\` destination, as long as the node is actually connected and the service is accessible.
- An SSH local port forward through a reachable home node is a useful fallback when direct access fails.
- Confirm each layer separately: LAN ping, TCP port, macOS routing, Tailscale peer ping, then application access.

### Follow-up: Home network redesign

**Planned after returning home; not yet implemented.**

Review moving devices off the flat \`192.168.1.0/24\` default LAN into the existing management, server, user, security, test, and guest VLANs. Also consider a less common address range for any remaining general-purpose LAN. Before making changes, inventory fixed IPs, AT&T gateway settings, Proxmox management, switch addresses, OPNsense routing, DNS, Tailscale subnet advertisements, firewall rules, and service dependencies. Migrate incrementally and keep a documented rollback path. Changing the default LAN by itself is not a guarantee against future subnet overlaps.
