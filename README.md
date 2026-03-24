# MPLS Network Automation and Testing

This project automates the configuration of MPLS on a Juniper SRX240 lab topology using Ansible and Jinja2 templates. It targets a multi-area OSPF setup with an MPLS core, and includes tasks to validate connectivity and collect routing state.

---

## Topology Overview

The lab consists of 6 Juniper SRX240 routers arranged across three OSPF areas with an MPLS-enabled core:

```
[srx240_1] --- area 1 --- [srx240_2] --- area 0 (MPLS core) --- [srx240_4] --- area 2 --- [srx240_6]
                                  \                              /
                               [srx240_3] -- [srx240_5]
```

| Router    | Management IP   | Role              | OSPF Areas  |
|-----------|-----------------|-------------------|-------------|
| srx240_1  | 192.168.1.29    | Edge (Area 1)     | Area 1      |
| srx240_2  | 192.168.1.30    | ABR / MPLS core   | Area 0, 1   |
| srx240_3  | 192.168.1.31    | MPLS core         | Area 0      |
| srx240_4  | 192.168.1.32    | ABR / MPLS core   | Area 0, 2   |
| srx240_5  | 192.168.1.33    | MPLS core         | Area 0      |
| srx240_6  | 192.168.1.34    | Edge (Area 2)     | Area 2      |

---

## Project Structure

```
.
├── inventory.yaml          # Host inventory with interface/IP/area definitions
├── project01.yaml          # Main Ansible playbook
├── interfaces.conf         # Jinja2 template — interface IP addressing
├── ospf_template.conf      # Jinja2 template — OSPF + security zones
└── mpls_template.conf      # Jinja2 template — LDP and MPLS on core interfaces
```

---

## Prerequisites

- Ansible with the [`Juniper.junos`](https://github.com/Juniper/ansible-junos-stdlib) role installed
- Python 3 (`/usr/bin/python3`) on the control node
- SSH/NETCONF access to all routers (credentials: `labuser` / `Labuser`)
- All routers reachable on their management IPs

Install the Juniper Ansible role:
```bash
ansible-galaxy install Juniper.junos
```

---

## Configuration Templates

### `interfaces.conf`
Configures IPv4 addresses on all interfaces for every router using values from the inventory.

### `ospf_template.conf`
- Configures OSPF per interface, assigning each to its defined area
- Sets `lo0` as a passive interface
- Adds all interfaces to the `trust` security zone with full `host-inbound-traffic` permissions

### `mpls_template.conf`
- Enables `family mpls` on `ge-0/0/1` and `ge-0/0/2` (core-facing interfaces)
- Configures LDP on `ge-0/0/1`, `ge-0/0/2`, and `lo0`
- Applied only to the MPLS core routers: `srx240_2` through `srx240_5`

---

## Running the Playbook

```bash
ansible-playbook -i inventory.yaml project01.yaml
```

### What the playbook does

**Play 1 — All routers:**
1. Pushes interface IP addressing from `interfaces.conf`
2. Pushes OSPF configuration and security zones from `ospf_template.conf`
3. Saves routing table output to `<hostname>-routing-information.txt`

**Play 2 — MPLS core routers (`srx240_2`–`srx240_5`) only:**
1. Pushes LDP and MPLS interface config from `mpls_template.conf`
2. Saves LDP neighbor state to `<hostname>-ldp-neighbors.txt`
3. Saves `inet.3` routes to `<hostname>-inet3-routes.txt`
4. Saves `mpls.0` routes to `<hostname>-mpls-routes.txt`

**Play 3 — Connectivity test from `srx240_1`:**
1. Pings `9.1.1.2` (the edge interface on `srx240_6`) to validate end-to-end reachability across the MPLS core
2. Prints the ping result to stdout

---

## Output Files

After a successful run, the following files are generated in the playbook directory:

| File | Description |
|------|-------------|
| `<host>-routing-information.txt` | Full routing table (all routers) |
| `<host>-ldp-neighbors.txt` | LDP neighbor adjacencies (core only) |
| `<host>-inet3-routes.txt` | MPLS label-switched paths via inet.3 (core only) |
| `<host>-mpls-routes.txt` | MPLS forwarding table mpls.0 (core only) |

---

## Inventory Structure

Interfaces are defined per host in `inventory.yaml` using a list of dictionaries:

```yaml
srx240_3:
  ansible_host: 192.168.1.31
  interfaces:
    - {if_name: "ge-0/0/1", if_addr: "7.1.1.2/30", area: "0"}
    - {if_name: "ge-0/0/2", if_addr: "7.1.1.5/30", area: "0"}
    - {if_name: "lo0",      if_addr: "1.1.1.3/24",  area: "0"}
```

Each entry requires:
- `if_name` — Junos interface name
- `if_addr` — IP address with prefix length
- `area` — OSPF area number

---

## Notes

- All configurations are pushed using `load: merge`, so existing config outside the managed stanzas is preserved.
- MPLS is only enabled on interfaces named `ge-0/0/1` or `ge-0/0/2` — the template filters by interface name.
- The loopback (`lo0`) is included in LDP but not in the MPLS interface list, which is standard practice for LDP router-ID advertisement.
