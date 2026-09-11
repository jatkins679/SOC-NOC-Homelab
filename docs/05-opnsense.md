# OPNsense Routing and Firewalling

## Current Deployment

`fw01` is OPNsense VM 220 on `pve01`. It connects the existing flat/home network to the segmented lab and routes between lab VLANs.

| Interface | OPNsense name | Network | Purpose |
|---|---|---|---|
| Flat side | WAN | `192.168.1.187/24` | Connection to the AT&T/TRENDnet network |
| `lan` | LABMGMT | `10.10.10.0/24` | Management; gateway `10.10.10.1` |
| `opt1` | SERVERS | `10.10.20.0/24` | Server VLAN |
| `opt2` | USERS | `10.10.30.0/24` | User VLAN |
| `opt3` | SOCNOC | `10.10.40.0/24` | Monitoring/SOC VLAN |
| `opt4` | RED | `10.10.50.0/24` | Attack-lab VLAN |
| `opt5` | DMZRANGE | `10.10.60.0/24` | Vulnerable-target VLAN |

## Rule Model

OPNsense filters a routed connection on the interface where the traffic enters. Rules must therefore be created on the source interface and must identify the real source, destination, protocol, and destination port.

For connections sourced on the flat `192.168.1.0/24` network and routed through OPNsense to a VLAN, enable **Disable reply-to** on the trusted WAN rule when the source's default gateway remains `192.168.1.254`. This prevents an asymmetric return path through the AT&T gateway.

Validated examples include:

- USERS `10.10.30.160` to Wazuh `10.10.40.20` TCP 1514-1515;
- DMZRANGE `10.10.60.10` to Wazuh `10.10.40.20` TCP 1514-1515;
- John's Mac `192.168.1.77` to `target01` `10.10.60.10` TCP 22, with Disable reply-to;
- intentionally authorized RED-to-target attack-lab traffic.

The complete rulebase is managed through OPNsense configuration backups/exports. CSV rule import adds or updates interface rules; the total rules page also counts automatically generated rules, so its displayed count is not the number of imported CSV rows.

## GUI Access from John's Mac

Run this on **John's Mac** and leave the terminal open:

```bash
ssh -N -L 8443:10.10.10.1:443 root@192.168.1.10
```

Open `https://localhost:8443` in a browser.

## Validation

Validate every rule from the actual source host. If a permitted TCP connection fails, use OPNsense live firewall logs and packet capture to confirm SYN, SYN-ACK, and the return path. A green/pass log entry alone does not prove the end-to-end session completed.

## Open Audit Items

GitHub issue #1 tracks duplicate rules, broad rules, monitoring/remote-access dependencies, DNS gaps, and removal of legacy addresses. Follow [`24-vlan-migration-dependency-checklist.md`](24-vlan-migration-dependency-checklist.md) before moving another asset.
