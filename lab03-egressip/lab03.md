# Lab 03: EgressIP and Dual-Spine ECMP Load Balancing

## Overview

This lab demonstrates two key concepts: EgressIP for stable egress identity, and dual-spine ECMP for redundant cross-cluster routing. Two c8000v routers (same AS 64514, iBGP) sit on separate L2 segments, with each cluster node dual-homed to both. This gives FRR on each node two equal-cost BGP paths for cross-cluster traffic.

## What You'll Learn

- Default CUDN egress behavior (SNAT to node IP)
- Applying EgressIP CRs to pin egress source addresses
- How EgressIP interacts with CUDNs (external egress only, not cross-cluster)
- Dual-spine topology with iBGP between two c8000v routers
- ECMP with `maximum-paths` and `maximum-paths ibgp` on the c8000v
- Verifying load distribution across BGP paths using CEF
- NMState NNCPs for second NIC IP configuration

## Network Design

```
                 lab-network (172.16.252.0/24)
                 +-----------------+
                 |    c8000v-1     |
                 |  AS 64514       |
                 |  .50            |
                 |  ECMP: 3 paths  |
                 +--------+--------+
                          |                     .70 (EgressIP)
   .61     .62     .63    |    .64
  +----+ +----+ +----+   |   +----+
  | M1 | | M2 | | M3 |---+---| W1 |   <-- lab-network NIC
  +----+ +----+ +----+       +----+
  | M1 | | M2 | | M3 |-------| W1 |   <-- lab-network-253 NIC
  +----+ +----+ +----+       +----+
   .61     .62     .63    |    .64
                          |
                 +--------+--------+
                 |    c8000v-2     |
                 |  AS 64514       |
                 |  .50            |
                 |  ECMP: 3 paths  |
                 +-----------------+
                 lab-network-253 (172.16.253.0/24)

  Hub AS 64512                  Spoke AS 64513
  CUDN: 10.100.0.0/16          CUDN: 10.200.0.0/16
  EgressIP: 172.16.252.70

  iBGP: c8000v-1 (172.16.252.50) <-> c8000v-2 (172.16.252.51)
         via lab-network, next-hop-self
```

## iBGP Design

Both c8000v routers share AS 64514 and peer via iBGP on `lab-network` (172.16.252.50 <-> 172.16.252.51). Each router uses `next-hop-self` so its iBGP peer can reach routes learned from the clusters. `maximum-paths ibgp 2` allows both the direct eBGP path and the iBGP-learned path to be used simultaneously.

## Prerequisites

- Hub and spoke clusters deployed (OCP 4.20+)
- `ansible-playbook common/setup.yaml` completed (creates both VMs and second NIC)

## Automated Deployment

```bash
# 1. Deploy infrastructure (NMState, NNCPs, CUDNs, BGP, ECMP, iBGP)
ansible-playbook lab03-egressip/lab03.yaml

# 2. Run four-phase verification
ansible-playbook lab03-egressip/verify.yaml

# Individual phases
ansible-playbook lab03-egressip/verify.yaml -t phase1   # Default egress
ansible-playbook lab03-egressip/verify.yaml -t phase2   # EgressIP applied
ansible-playbook lab03-egressip/verify.yaml -t phase3   # ECMP on c8000v-1
ansible-playbook lab03-egressip/verify.yaml -t phase4   # Dual-spine verification
```

## Key Learnings

### EgressIP only affects external egress

EgressIP SNATs traffic leaving the cluster to external destinations. Cross-cluster CUDN traffic (hub pod -> spoke pod via BGP) is NOT affected -- it uses the pod's CUDN IP as the source, routed through the BGP path.

### Dual-spine ECMP

With two c8000v routers, each cluster node has two BGP peers for cross-cluster routes. FRR on each node receives the same prefixes from both routers, providing two equal-cost paths. If one router fails, traffic fails over to the other.

### ECMP distributes return traffic

With `maximum-paths 3` on each c8000v, return traffic to 10.100.0.0/16 is distributed across all 3 hub nodes using per-destination hashing. `maximum-paths ibgp 2` adds the iBGP-learned paths as additional ECMP entries.

### NIC identification via MAC address

The second NIC added to cluster VMs gets a predictable MAC address (set via `virsh attach-interface --mac`). NMState NNCPs use `identifier: mac-address` to match and configure the interface regardless of its kernel name.

## Files Reference

| File | Description |
|------|-------------|
| `lab03.yaml` | Deploy NMState, NNCPs, CUDNs, BGP, ECMP, and iBGP |
| `verify.yaml` | Four-phase EgressIP and dual-spine ECMP demonstration |
| `manifest/hub-egressip.yaml` | EgressIP CR (172.16.252.70 for cudn-hub-ns) |
| `manifest/nncp-spine2.yaml` | NNCPs for hub node second NIC IPs |
| `manifest/spoke-nncp-spine2.yaml` | NNCP for spoke worker second NIC IP |
| `manifest/hub-cudn.yaml` | Hub namespace + ClusterUserDefinedNetwork |
| `manifest/hub-bgp.yaml` | Hub FRRConfiguration (2 neighbors) + RouteAdvertisements |
| `manifest/spoke-cudn.yaml` | Spoke namespace + ClusterUserDefinedNetwork |
| `manifest/spoke-bgp.yaml` | Spoke FRRConfiguration (2 neighbors) + RouteAdvertisements |
