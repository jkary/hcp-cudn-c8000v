# Lab 04: VRF Lite with VLAN Trunking

## Overview

This lab creates a VRF-based multi-tenant isolation setup where three tenants share the same physical infrastructure but have isolated routing domains. VLAN trunking between OCP nodes and the c8000v provides L2 separation. Tenant C demonstrates selective route leaking — it can reach both tenant A and B, while A and B remain isolated from each other. This is the real-world pattern that leads to EVPN/VXLAN.

## What You'll Learn

- Configuring VLAN sub-interfaces on OCP nodes via NMState (NodeNetworkConfigurationPolicy)
- Configuring VRF definitions and sub-interfaces on c8000v
- Per-VRF BGP address-families on the router
- Selective VRF route leaking via route-target import/export
- Creating isolated CUDNs per tenant with separate FRR peering over dedicated VLANs
- Proving tenant isolation: tenant A pods cannot reach tenant B pods
- Inspecting VRF routing tables (`show ip route vrf`, `show ip vrf`)

## Network Design

```
Hub Nodes (enp3s0 trunk)              c8000v (Gi1 trunk)           Spoke Worker
┌─────────────────────┐               ┌──────────────────┐        ┌────────────┐
│ enp3s0.100 (VLAN 100)│──────────────│ Gi1.100 TENANT-A │        │            │
│   172.16.100.6x/24  │  Tenant A    │   172.16.100.50  │        │            │
│                      │              │                   │        │            │
│ enp3s0.300 (VLAN 300)│──────────────│ Gi1.300 TENANT-C │        │            │
│   172.16.150.6x/24  │  Tenant C    │   172.16.150.50  │        │            │
│                      │              │                   │        │ enp3s0.200 │
│                      │              │ Gi1.200 TENANT-B │────────│ VLAN 200   │
│                      │              │   172.16.200.50  │  T-B   │172.16.200. │
│                      │              │                   │        │  64/24     │
│ enp3s0 (untagged)    │──────────────│ Gi1 (mgmt)       │────────│ enp3s0     │
│   172.16.252.6x/24  │              │   172.16.252.50  │        │172.16.252. │
└─────────────────────┘               └──────────────────┘        │  64/24     │
                                                                   └────────────┘
```

| Component | Cluster | VLAN | Subnet | CUDN CIDR | VRF |
|-----------|---------|------|--------|-----------|-----|
| Tenant A | Hub | 100 | 172.16.100.0/24 | 10.100.0.0/16 | TENANT-A |
| Tenant B | Spoke | 200 | 172.16.200.0/24 | 10.200.0.0/16 | TENANT-B |
| Tenant C | Hub | 300 | 172.16.150.0/24 | 10.150.0.0/16 | TENANT-C |

### VRF Route-Target Policy

| VRF | Export RT | Import RT | Effect |
|-----|----------|-----------|--------|
| TENANT-A | 64514:100 | 64514:100, 64514:300 | Sees A + C routes |
| TENANT-B | 64514:200 | 64514:200, 64514:300 | Sees B + C routes |
| TENANT-C | 64514:300 | 64514:100, 64514:200, 64514:300 | Sees all routes |

Tenant C can reach both A and B. Tenants A and B remain isolated from each other.

## Prerequisites

- Hub cluster deployed (OCP 4.20+)
- `ansible-playbook common/setup.yaml` completed
- NMState operator will be installed automatically by the playbook

## Automated Deployment

```bash
# Deploy everything (NMState, VLANs, VRFs, CUDNs, BGP)
ansible-playbook lab04-vrf-lite/lab04.yaml

# Verify isolation and connectivity
ansible-playbook lab04-vrf-lite/verify.yaml
```

## Key Learnings

### VLAN trunking provides L2 isolation

Each tenant gets a dedicated VLAN between the OCP nodes and the c8000v. The libvirt bridge passes tagged frames (vlan_filtering=0). NMState creates the VLAN sub-interfaces declaratively on each node.

### VRF route-target leaking enables selective connectivity

The c8000v uses route-target import/export to control which VRFs see each other's routes. Tenant C imports route-targets from both A and B, giving it full visibility. A and B only import from C (and themselves), keeping them isolated from each other.

### Cross-VRF data-plane limitation

The c8000v control plane correctly leaks routes between VRFs via route-targets. However, cross-tenant pod connectivity (e.g., tenant C pinging tenant A) requires FRR-K8s to resolve next-hops across Linux VRF boundaries on the OCP nodes. Currently, FRR-K8s cannot install routes with cross-VRF next-hops, so the route leaking is visible on the c8000v routing tables but not yet functional at the pod data-plane level. This is an area for future improvement as EVPN/VXLAN matures in the OCP networking stack.

### This lab replaces the c8000v BGP config

Unlike other labs which add to the base BGP config, lab04 removes `router bgp 64514` and creates a new BGP process with per-VRF address-families. Run `common/teardown.yaml` and `common/setup.yaml` to restore the base config after this lab.

### Path to EVPN

This VRF lite setup is the foundation for EVPN/VXLAN. The next step would be replacing the VLAN-based L2 separation with VXLAN tunnels and using MP-BGP EVPN for control-plane signaling, eliminating the need for per-tenant VLANs.

## Files Reference

| File | Description |
|------|-------------|
| `lab04.yaml` | Deploy NMState, VLANs, VRFs, CUDNs, and BGP peering |
| `verify.yaml` | Test isolation (A/B) and connectivity (C->A, C->B) |
| `manifest/nmstate-operator.yaml` | NMState operator Subscription |
| `manifest/nmstate-instance.yaml` | NMState CR to activate |
| `manifest/nncp-vlans.yaml` | Hub VLAN sub-interfaces (VLAN 100, 300) |
| `manifest/spoke-nncp-vlans.yaml` | Spoke VLAN sub-interface (VLAN 200) |
| `manifest/hub-tenant-a-cudn.yaml` | Tenant A namespace + CUDN (hub, 10.100.0.0/16) |
| `manifest/hub-tenant-a-bgp.yaml` | Tenant A FRRConfiguration (VLAN 100) |
| `manifest/hub-tenant-c-cudn.yaml` | Tenant C namespace + CUDN (hub, 10.150.0.0/16) |
| `manifest/hub-tenant-c-bgp.yaml` | Tenant C FRRConfiguration (VLAN 300) |
| `manifest/spoke-tenant-b-cudn.yaml` | Tenant B namespace + CUDN (spoke, 10.200.0.0/16) |
| `manifest/spoke-tenant-b-bgp.yaml` | Tenant B FRRConfiguration (VLAN 200) |
| `manifest/c8000v-vrf.cfg` | c8000v VRF + sub-interfaces + BGP with route leaking |

## Depends On

- `common/setup.yaml`
- Note: This lab replaces the c8000v BGP config (VRF restructures the BGP process)
