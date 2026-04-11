# MPLS L3VPN Network Automation

This project automates deployment of a full **BGP-based MPLS L3VPN** on a Juniper SRX240 lab topology using Ansible and Jinja2 templates.

---

## Routing Design

| Plane | Protocol | Scope | Purpose |
|-------|----------|-------|---------|
| IGP | OSPF area 0 | PE + P routers only | Distributes loopback reachability so iBGP sessions can form |
| Label distribution | LDP | PE + P routers | Builds transport LSPs across the core |
| PE-PE | MP-BGP iBGP (inet-vpn) | PE1 ↔ PE2 via loopbacks | Exchanges VPN routes between PEs |
| PE-CE | eBGP (inside VRF) | PE ↔ CE per customer | Distributes customer prefixes into the VRF |

CE routers run no OSPF and have no MPLS awareness — they only speak eBGP toward their attached PE.

---

## Topology

```
[CE1: srx240_1]                                              [CE2: srx240_6]
  AS 65001                                                     AS 65002
     |  eBGP (8.1.1.0/30)                  eBGP (9.1.1.0/30)  |
[PE1: srx240_2] ======== MPLS core (OSPF area 0) ======= [PE2: srx240_4]
  AS 65000                /                    \               AS 65000
              [P: srx240_3]          [P: srx240_5]
              
              <-------- iBGP MP-BGP (inet-vpn) -------->
                         PE1 lo 1.1.1.2 ↔ PE2 lo 1.1.1.4
```

| Router    | Mgmt IP       | Role | AS    | OSPF | VRF |
|-----------|---------------|------|-------|------|-----|
| srx240_1  | 192.168.1.29  | CE   | 65001 | —    | —   |
| srx240_2  | 192.168.1.30  | PE   | 65000 | area 0 | CUSTOMER_A |
| srx240_3  | 192.168.1.31  | P    | —     | area 0 | — |
| srx240_4  | 192.168.1.32  | PE   | 65000 | area 0 | CUSTOMER_A |
| srx240_5  | 192.168.1.33  | P    | —     | area 0 | — |
| srx240_6  | 192.168.1.34  | CE   | 65002 | —    | —   |

**Customer A VRF:** RD `65000:100` | RT import/export `65000:100`

---

## Project Structure

```
.
├── inventory.yaml          # Hosts, interfaces, VRF, and BGP definitions
├── project01.yaml          # Main Ansible playbook (9 plays)
├── interfaces.conf         # Jinja2 — IP addressing (all routers)
├── ospf_template.conf      # Jinja2 — OSPF area 0 + security zones (PE + P only)
├── mpls_template.conf      # Jinja2 — LDP + MPLS on core interfaces (PE + P)
├── bgp_template.conf       # Jinja2 — MP-BGP iBGP with inet-vpn family (PE only)
├── vrf_template.conf       # Jinja2 — VRF instance + eBGP CE neighbor (PE only)
└── ce_bgp_template.conf    # Jinja2 — eBGP toward PE (CE only)
```

---

## Running the Playbook

```bash
ansible-playbook -i inventory.yaml project01.yaml
```

### Playbook Summary (9 plays)

| Play | Hosts | What it does |
|------|-------|--------------|
| 1 | all | Interface IP addressing |
| 2 | mpls_core | OSPF area 0 on core + loopback interfaces only |
| 3 | mpls_core | MPLS family + LDP on core interfaces |
| 4 | pe_routers | MP-BGP iBGP (inet-vpn unicast) between PE1 and PE2 |
| 5 | pe_routers | VRF routing-instance + eBGP CE neighbor inside VRF |
| 6 | ce_routers | eBGP toward attached PE |
| 7 | mpls_core | Collect routing, LDP, inet.3, mpls.0 state |
| 8 | pe_routers | Collect BGP, bgp.l3vpn.0, CUSTOMER_A.inet.0 state |
| 9 | srx240_2 + srx240_1 | VRF ping (PE1→CE2 lo) + CE1→CE2 end-to-end ping |

---

## Output Files

| File | Source |
|------|--------|
| `<host>-routing-information.txt` | Full routing table (core) |
| `<host>-ldp-neighbors.txt` | LDP adjacencies (core) |
| `<host>-inet3-routes.txt` | LDP LSPs in inet.3 (core) |
| `<host>-mpls-routes.txt` | MPLS forwarding table mpls.0 (core) |
| `<host>-bgp-neighbors.txt` | MP-BGP neighbor state (PE) |
| `<host>-l3vpn-routes.txt` | bgp.l3vpn.0 VPN routes (PE) |
| `<host>-customer-a-vrf-routes.txt` | CUSTOMER_A.inet.0 VRF table (PE) |

---

## L3VPN Data Plane

```
CE1 sends packet → PE1 (ge-0/0/3, VRF CUSTOMER_A)
  PE1 looks up CUSTOMER_A.inet.0, finds next-hop = PE2 loopback
  PE1 pushes VPN label (from bgp.l3vpn.0) + LDP transport label (from inet.3)
  P routers swap transport label across the core
  PE2 pops transport label (PHP or explicit-null), then pops VPN label
  PE2 looks up CUSTOMER_A.inet.0, forwards out ge-0/0/3 to CE2
```

---

## Notes

- OSPF is scoped strictly to `type: core` and `type: loopback` interfaces in the inventory — the PE-CE interface (`ge-0/0/3`) is intentionally excluded from OSPF to keep customer routing isolated.
- PE-CE eBGP sessions live **inside the VRF** (`routing-instances { CUSTOMER_A { protocols { bgp } } }`), not in the global routing table.
- `vrf-table-label` allocates a per-VRF label for packets arriving from the core, enabling correct VRF lookup on the egress PE.
- To add a second customer, add a new VRF block with a distinct RD/RT (e.g. `65000:200`) and a new CE AS, then assign its PE-facing interface.
