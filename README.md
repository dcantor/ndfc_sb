# ndfc_sb — Nexus 9000 core routers via NDFC (BGP + VRF Lite, no VXLAN)

Ansible project that deploys **four** Cisco Nexus 9000 **data-center core routers**
(two per data center, across two DCs) through **Cisco NDFC** (Nexus Dashboard Fabric
Controller) using the `cisco.dcnm` collection.

## Topology

```
        DC1 (AS 65001)                         DC2 (AS 65002)
   ┌───────────────────┐                  ┌───────────────────┐
   │  dc1-core1 ═══════ dc1-core2          dc2-core1 ═══════ dc2-core2  │
   │      │   (2 links, iBGP)                  (2 links, iBGP)   │      │
   └──────┼────────────────────┘          └──────────┼─────────┘
          │                                           │
          └──────────── eBGP (DCI, square) ───────────┘
         dc1-core1↔dc2-core1   and   dc1-core2↔dc2-core2
```

- **2 devices per DC**, with **2 links** between the pair, **iBGP** peered.
- **eBGP between the data centers**, "square" DCI (2 links: core1↔core1, core2↔core2).
- **BFD for BGP on every link** (and every BGP neighbor).
- **Two VRFs via VRF Lite, fully isolated**, extended end-to-end across both DCs.
- **No EVPN / no VXLAN** — plain IP routing only.

## Design

- **Loopback-based BGP.** Global table peers on `loopback0`; each VRF peers on its own
  loopback (`loopback1`=VRF_A, `loopback2`=VRF_B).
- **Intra-DC loopback reachability = OSPF.** A single OSPF process (`UNDERLAY`) runs on the
  two intra-DC links in the global table **and** in a context per VRF, so every intra-DC
  iBGP loopback (global + per-VRF) is reachable dynamically, ECMP over both links. OSPF runs
  point-to-point with **`ip ospf bfd`** on each link. It does **not** run on the DCI.
- **Inter-DC loopback reachability = static routes.** The DCI eBGP is loopback-to-loopback
  (`ebgp-multihop 2`), reached by a static route per peer over the DCI link.
- **BFD everywhere.** `feature bfd`, a `bfd interval` on every L3 link and sub-interface,
  and `bfd` on every BGP neighbor.
- **VRF isolation.** Each VRF has its own loopback, its own dot1q sub-interfaces, its own
  RD, and **no `route-target import/export`** — so the two VRFs never exchange routes.

### Why these NDFC choices

- **Fabric type = `External`** — the correct light-touch NDFC fabric for managed
  routed/core devices that are *not* part of a VXLAN EVPN fabric. (`BGP` fabric_type is an
  eBGP-routed VXLAN underlay; `LAN_CLASSIC` is L2 access/aggregation — both wrong here.)
- **`dcnm_vrf` / `dcnm_network` are VXLAN-EVPN-only and are NOT used.** On an External
  fabric **all device config** — interfaces, loopbacks, sub-interfaces, BFD, VRF contexts,
  static routes and BGP — is delivered as a single **`switch_freeform` policy per device**
  (`dcnm_policy`), rendered from [templates/device_freeform.j2](templates/device_freeform.j2).
  This is more reliable than wrangling `dcnm_interface`/`dcnm_links` for a topology this size.
- **Switches are not SSH inventory hosts.** The only Ansible inventory host is the NDFC
  controller (`ansible.netcommon.httpapi`); modules fan out to switches by mgmt IP / serial.

## Layout

```
ansible.cfg                    inventory path, conn timeouts
requirements.yml               cisco.dcnm + ansible.netcommon (pinned)
inventory/hosts.yml            single host: the NDFC controller
group_vars/all.yml             fabric_name, fabric_bgp_as, default_mtu
group_vars/ndfc.yml            httpapi connection vars (+ login_domain)
vault/secrets.yml(.example)    NDFC creds + device creds (real file gitignored)
fabric_vars/core_fabric.yml    External fabric definition
vars/topology.yml              ★ single source of truth: 4 devices, ASNs, all links/sub-ints
vars/vrfs.yml                  the 2 isolated VRFs (loopback id, RD, networks, route-maps)
vars/bgp.yml                   BFD timers, eBGP multihop TTL, OSPF process/area
templates/device_freeform.j2   ★ generates each device's full CLI from the topology
playbooks/00_create_fabric     dcnm_fabric  → External fabric
playbooks/01_add_inventory     dcnm_inventory → 4 cores as core_router
playbooks/02_config            dcnm_policy → per-device freeform (the workhorse)
playbooks/03_deploy            confirm/query deployed policies
playbooks/site.yml             imports 00..03 in order
```

To change the topology, **edit `vars/topology.yml`** — the per-device CLI is generated
from it, so addresses/links/ASNs live in exactly one place.

## Prerequisites

- A reachable controller VIP, either:
  - **NDFC 12.2.x** on Nexus Dashboard 2.x/3.x, or
  - **Nexus Dashboard 4.x** (unified ND — Fabric Controller is a persona of ND, not a
    separate 12.x service). Validated target ND 4.1.1g via legacy APIs; requires
    `cisco.dcnm >= 3.9.0` and `ansible_httpapi_login_domain` set.
- All four N9Ks onboarded to ND/NDFC reachability (mgmt IP + credentials).
- `ansible-core >= 2.15`, Python `requests`.
- `ansible-galaxy collection install -r requirements.yml`

## Configure

1. Fill in `vars/topology.yml` (devices, mgmt IPs, serials, loopbacks, link IPs, ASNs),
   `vars/vrfs.yml`, `vars/bgp.yml`, and `fabric_vars/core_fabric.yml`.
2. Set the controller address in `inventory/hosts.yml` / `group_vars/ndfc.yml`.
3. Create your secrets file from the template and encrypt it:
   ```
   cp vault/secrets.yml.example vault/secrets.yml
   ansible-vault encrypt vault/secrets.yml
   ansible-vault edit vault/secrets.yml
   ```
   `vault/secrets.yml` is **gitignored** — only the `.example` template is committed.

## Run

```
ansible-galaxy collection install -r requirements.yml

ansible-playbook playbooks/site.yml --syntax-check
ansible-playbook playbooks/site.yml --check --ask-vault-pass     # dry run vs NDFC
ansible-playbook playbooks/site.yml --ask-vault-pass             # deploy
```

Individual stages can be run on their own (`playbooks/00_create_fabric.yml`, etc.).

## Verify after deploy

- In NDFC: fabric created, all four cores in `core_router` role, freeform policies
  attached, deploy succeeded.
- On a switch:
  - `show ip ospf neighbors` / `show ip ospf neighbors vrf VRF_A` — OSPF adjacencies up on
    both intra-DC links (global + per VRF); `show bfd neighbors` — OSPF and BGP BFD sessions Up.
  - `show ip route 10.255.1.2` — peer `loopback0` learned via OSPF, ECMP over both intra links.
  - `show ip bgp summary` — intra-DC iBGP (global) up.
  - `show ip bgp vrf VRF_A summary` / `vrf VRF_B` — per-VRF iBGP (intra-DC) and eBGP
    (inter-DC) up.
  - `show running-config | section 'vrf context'` — **no** `route-target import/export`
    (isolation confirmed).

## Risks / notes

1. `cisco.dcnm` version-sensitivity: `fabric_type` casing (`External`) is version-sensitive
   — verify on your release.
   - **Nexus Dashboard 4.x:** there is no "NDFC 4.x" — ND 4.x is the converged platform that
     absorbs the Fabric Controller. `cisco.dcnm` drives it via the **legacy API** layer
     (added in collection 3.9.0); the playbooks work unchanged provided you use `>=3.9.0`
     and set `ansible_httpapi_login_domain`. ND 4.0 is pre-migration; 4.1.1g is validated.
2. The External fabric is light-touch: NDFC stores/deploys freeform but does **not** validate
   BGP correctness. Policy `description` keys are kept stable so re-runs stay idempotent.
3. **BFD on loopback (multihop) sessions:** the link-level `bfd interval` gives single-hop
   BFD on each transit link, and `neighbor … bfd` enables BFD for the session. On some NX-OS
   platforms multihop BFD for BGP additionally needs `bfd multihop` enabled — verify on your
   hardware. Fast intra-DC reconvergence also comes from the ECMP static pair.
4. AS numbers (65001/65002), loopbacks and link subnets in `vars/topology.yml` are lab
   placeholders — replace with your addressing.
