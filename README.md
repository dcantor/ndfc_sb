# ndfc_sb — Nexus 9000 core routers via NDFC (BGP + VRF Lite, no VXLAN)

Ansible project that deploys a pair of Cisco Nexus 9000 **data-center core routers**
through **Cisco NDFC** (Nexus Dashboard Fabric Controller) using the `cisco.dcnm`
collection.

## Design

- **No EVPN / no VXLAN** — plain IP routing only.
- Everything is driven **through NDFC** (`cisco.dcnm`), not direct SSH/NX-API.
- **BGP:** single AS, **iBGP between the two cores**, **eBGP toward upstream/downstream peers**.
- **Two VRFs via VRF Lite, fully isolated** — no route leaking between them.

### Why these choices

- **Fabric type = `External`.** This is the correct light-touch NDFC fabric for managed
  routed/core devices that are *not* part of a VXLAN EVPN fabric. (`BGP` fabric_type is an
  eBGP-routed VXLAN underlay; `LAN_CLASSIC` is L2 access/aggregation — both wrong here.)
- **`dcnm_vrf` / `dcnm_network` are VXLAN-EVPN-only and are deliberately NOT used.** On an
  External fabric all `vrf context` + `router bgp` config is delivered as **freeform switch
  policies via `dcnm_policy`**, rendered from the Jinja2 templates in `templates/`.
- **Switches are not SSH inventory hosts.** The only Ansible inventory host is the NDFC
  controller, reached via `ansible.netcommon.httpapi`. Modules fan out to switches by mgmt
  IP / serial.

### Routing model

- **iBGP** in the global address-family between the two cores (peer on loopbacks).
- **eBGP** inside each VRF's `address-family ipv4 unicast` toward the external peer.
- A **per-VRF iBGP session** between the two cores carries each VRF's external routes
  core-to-core without leaking into the global table.
- **Isolation:** each VRF has its own RD and dedicated sub-interface, and the `vrf context`
  templates contain **no `route-target import/export`**.

## Layout

```
ansible.cfg                  inventory path, conn timeouts
requirements.yml             cisco.dcnm + ansible.netcommon (pinned)
inventory/hosts.yml          single host: the NDFC controller
group_vars/all.yml           fabric_name, core_bgp_as, common defaults
group_vars/ndfc.yml          httpapi connection vars
vault/secrets.yml            NDFC creds + per-switch device creds (ansible-vault)
fabric_vars/core_fabric.yml  External fabric definition
host_vars/core1.yml          per-core: mgmt IP, serial, loopback, links, sub-int IPs
host_vars/core2.yml
vars/vrfs.yml                the 2 isolated VRFs
vars/bgp.yml                 AS, router-ids, peer tables
templates/*.j2               freeform CLI: vrf context, bgp global, per-VRF AF
playbooks/00..05 + site.yml  ordered deployment
```

## Prerequisites

- A reachable controller VIP, either:
  - **NDFC 12.2.x** on Nexus Dashboard 2.x/3.x, or
  - **Nexus Dashboard 4.x** (unified ND — the Fabric Controller is now a persona of ND,
    not a separate 12.x service). Validated target is ND 4.1.1g via the legacy APIs.
    Requires `cisco.dcnm >= 3.9.0` and `ansible_httpapi_login_domain` set (see below).
- The two N9K cores already onboarded to ND/NDFC reachability (mgmt IP + credentials).
- `ansible-core >= 2.15`, Python `requests`.
- `ansible-galaxy collection install -r requirements.yml`

## Configure

1. Fill in `vars/vrfs.yml`, `vars/bgp.yml`, `host_vars/core1.yml`, `host_vars/core2.yml`,
   and `fabric_vars/core_fabric.yml` with your real topology.
2. Set the controller address in `inventory/hosts.yml` / `group_vars/ndfc.yml`.
3. Create your secrets file from the template and encrypt it:
   ```
   cp vault/secrets.yml.example vault/secrets.yml
   ansible-vault encrypt vault/secrets.yml
   ansible-vault edit vault/secrets.yml
   ```
   `vault/secrets.yml` is **gitignored** — only the `.example` template is committed,
   so real credentials are never pushed.

## Run

```
# Install collections
ansible-galaxy collection install -r requirements.yml

# Validate
ansible-playbook playbooks/site.yml --syntax-check
ansible-playbook playbooks/site.yml --check --ask-vault-pass     # dry run vs NDFC

# Deploy
ansible-playbook playbooks/site.yml --ask-vault-pass
```

Individual stages can be run on their own (`playbooks/00_create_fabric.yml`, etc.).

## Verify after deploy

- In NDFC: fabric created, both cores in `core_router` role, sub-interfaces + VRF-Lite
  links present, freeform policies attached, deploy succeeded.
- On a switch:
  - `show ip bgp summary` — iBGP up between cores.
  - `show ip bgp vrf VRF_A summary` / `vrf VRF_B` — eBGP up per VRF.
  - `show running-config | section 'vrf context'` — **no** `route-target import/export`
    (isolation confirmed).

## Risks / notes

1. `cisco.dcnm` 3.x assumed. `fabric_type` casing (`External`) and link template names
   (`ext_fabric_setup`) are version-sensitive — verify on your release.
   - **Nexus Dashboard 4.x:** there is no "NDFC 4.x" — NDFC was 12.x; ND 4.x is the
     converged platform that absorbs the Fabric Controller. The `cisco.dcnm` modules
     still drive it through the **legacy API** layer (added in collection 3.9.0), so the
     playbooks here work unchanged *provided* you bump the collection (`>=3.9.0`) and set
     `ansible_httpapi_login_domain`. ND 4.0 specifically is pre-migration/early; 4.1.1g is
     the validated target. For ND platform-level onboarding you may also want the companion
     `cisco.nd` collection.
2. The External fabric is light-touch: NDFC stores/deploys freeform but does **not** validate
   BGP correctness. Policy `description` keys are kept stable so re-runs stay idempotent.
3. Upstream/downstream eBGP peers are assumed external to this fabric. If they are also
   NDFC-managed N9Ks, they need their own fabric + `ext_fabric_setup` links on the far end.
