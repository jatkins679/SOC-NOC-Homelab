# Ethernet Cabling and Labeling Standard

## Purpose

This document defines the Ethernet labeling and physical cable-record standard
for the SOC/NOC homelab.

The objective is simple: a disconnected cable should be identifiable without
tracing an unlabeled wire through the shelf or relying on memory.

---

# 1. Cable ID Format

Every permanent/semi-permanent Ethernet patch cable receives a unique ID:

```text
C01
C02
C03
...
```

The `C` identifies a cable and the two-digit number makes the IDs easy to sort,
read, and extend.

Cable IDs are physical identifiers. A cable keeps its `C##` number unless it is
retired or deliberately relabeled in the documentation.

---

# 2. Label Both Ends

Each Ethernet cable is labeled at **both ends**.

The label should identify:

1. cable ID;
2. endpoint/device name;
3. switch and switch port where useful.

Example endpoint-side label:

```text
C02 PVE01
```

Example switch-side record:

```text
C02 PVE01
SW01 Gi1
```

The physical label template intentionally included a blank `SW01 Gi__` field so
port assignments can be written at connection time.

---

# 3. Prepared Cable IDs

The current prepared label set includes:

| Cable ID | Endpoint | Initial switch-port state |
|---|---|---|
| `C01` | BGW320 / upstream | `SW01 Gi8` accepted convention |
| `C02` | `PVE01` | `SW01 Gi1` accepted convention |
| `C03` | `PVE02` | `SW01 Gi2` accepted convention |
| `C04` | `PVE03` | `SW01 Gi3` accepted convention |
| `C05` | `PVE04` | `SW01 Gi4` accepted convention |
| `C06` | `STORAGE01` | Record during rebuild |
| `C07` | `MGMT01` | Record during rebuild |
| `C08` | `DNS01` | Record during rebuild |

The port mapping for `C06`-`C08` is deliberately left to the physical rebuild
rather than invented in advance.

---

# 4. Endpoint Naming

Use infrastructure names that match the repository:

```text
PVE01
PVE02
PVE03
PVE04
STORAGE01
MGMT01
DNS01
SW01
BGW320
```

Avoid vague labels such as:

```text
SERVER
LAPTOP
PI
SWITCH
```

Those labels become ambiguous as the lab grows.

---

# 5. NIC Labels

Where a system has multiple Ethernet interfaces, label the **device/NIC** as well
as the cable.

This is especially important for Proxmox systems when one interface is used for
existing management/home connectivity and another is later introduced for lab
VLAN/trunk work.

A NIC label should describe the intended role, for example:

```text
MGMT
LAB
TRUNK
```

Do not commit MAC addresses or other unnecessary hardware identifiers to the
public repository merely to distinguish NICs.

---

# 6. Cable Length Standard

Use the shortest cable that reaches comfortably without tension.

For same-shelf devices:

- prefer short patch cables;
- avoid large coils of unused Ethernet cable;
- leave enough slack to remove or service a device;
- do not create tension at the RJ45 connector.

Longer runs should be reserved for actual shelf-to-shelf or wall-to-shelf paths.

---

# 7. Cable Management

Use reusable hook-and-loop/Velcro ties for cable bundles.

Reasons:

- easy to reopen during troubleshooting;
- less risk of crushing cable jackets than overtightened plastic zip ties;
- easy to add/remove one cable;
- supports repeated lab changes.

Plastic zip ties may be useful for permanent non-data anchoring, but they should
not be the default for frequently changed Ethernet bundles.

---

# 8. Switch-Port Record

After the physical rebuild, maintain a simple authoritative table:

| Port | Cable | Endpoint | Purpose / VLAN state |
|---|---|---|---|
| `Gi1` | `C02` | `pve01` | Flat/default initially; future role documented later |
| `Gi2` | `C03` | `pve02` | Flat/default initially; future role documented later |
| `Gi3` | `C04` | `pve03` | Flat/default initially; future role documented later |
| `Gi4` | `C05` | `pve04` | Flat/default initially; future role documented later |
| `Gi8` | `C01` | BGW320 / upstream | Initial upstream convention |

Add `C06`-`C08` only after their actual ports are connected and verified.

---

# 9. Change Procedure

When moving a cable:

1. identify the `C##` label;
2. record the current switch port;
3. disconnect only the intended cable;
4. connect it to the new intended port;
5. update the port record immediately;
6. verify link and endpoint connectivity;
7. update VLAN/access/trunk notes if the logical role changed.

Do not wait until the end of a large recabling job to reconstruct the port map
from memory.

---

# 10. Documentation and Evidence

Useful evidence for the repository:

- a sanitized switch-port map;
- a photo showing the `C##` label convention;
- a photo of the completed cable bundle/route;
- before/after shelf photographs;
- the logical link between cable IDs and `sw01` ports.

Do not publish passwords, credentials, private configuration backups, or other
sensitive information.

---

# 11. Skills Demonstrated

This standard demonstrates:

- physical-layer documentation;
- structured cable identification;
- switch-port mapping;
- endpoint naming discipline;
- serviceability planning;
- change tracking;
- troubleshooting readiness;
- separation of physical and logical network state.
