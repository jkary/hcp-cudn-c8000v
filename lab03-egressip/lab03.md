# Lab 03: EgressIP and Pod Identity

## Overview

By default, a pod's external traffic is SNAT'd to its node's IP — meaning the source IP changes whenever the pod is rescheduled. EgressIP pins a stable source IP to an entire namespace, giving workloads a consistent external identity. This lab demonstrates three identity scenarios that reveal the boundary between external and cross-cluster identity.

## What You'll Learn

- Default CUDN egress behavior (SNAT to node IP — unpredictable identity)
- Applying an EgressIP CR to pin a stable egress source address
- Using c8000v ACL logging to observe the source IP transformation
- Cross-cluster CUDN traffic preserves pod CUDN IPs (EgressIP does not apply)
- The identity boundary: EgressIP controls external identity, CUDN controls cross-cluster identity

## Network Design

```
  Hub Cluster (AS 64512)              c8000v (AS 64514)             Spoke (AS 64513)
  CUDN: 10.100.0.0/16                                              CUDN: 10.200.0.0/16
  EgressIP: 172.16.252.70

  +------+  +------+  +------+     +-------------------+           +------+
  |  M1  |  |  M2  |  |  M3  |     |      c8000v       |           |  W1  |
  |  .61 |  |  .62 |  |  .63 |     | Gi1: .50          |           |  .64 |
  +--+---+  +--+---+  +--+---+     | Gi2: .50          |           +--+---+
     |         |         |          +--+-------------+--+              |
     |         |         |             |             |                 |
  ---+---------+---------+-------------+---       ---+-----------------+---
       lab-network (172.16.252.0/24)                lab-network-253 (172.16.253.0/24)
```

## Three Identity Scenarios

### Phase 1: Default egress — node IP as source

Without EgressIP, when a hub pod pings the c8000v (172.16.252.50), OVN SNATs the traffic to the pod's node IP (172.16.252.61, .62, or .63). The c8000v's ACL log shows this node IP as the source. If the pod moves to a different node, the source IP changes — there is no stable identity.

### Phase 2: EgressIP — stable namespace identity

After applying the EgressIP CR (172.16.252.70 for `cudn-hub-ns`), the same pod's external traffic now exits with 172.16.252.70 as the source. The c8000v ACL log confirms this. Every pod in the namespace shares this identity, regardless of which node it runs on.

### Phase 3: Cross-cluster CUDN — pod identity preserved

When the hub pod pings a spoke pod over the CUDN BGP route, the source IP is the pod's CUDN IP (10.100.x.x), NOT 172.16.252.70. A tcpdump on the spoke pod proves this. EgressIP does not apply to cross-cluster CUDN traffic.

## Why EgressIP Doesn't Affect Cross-Cluster Traffic

EgressIP and cross-cluster CUDN traffic follow different routing paths through OVN:

```
EXTERNAL EGRESS (EgressIP applies):
  Pod -> OVN -> node external interface -> SNAT to 172.16.252.70 -> c8000v

CROSS-CLUSTER CUDN (EgressIP does NOT apply):
  Pod -> OVN CUDN overlay -> FRR -> BGP -> c8000v -> BGP -> FRR -> OVN -> remote pod
```

The difference is where OVN makes its routing decision. When a pod sends traffic to an external destination (like the c8000v management IP), OVN routes it out the node's external interface and applies EgressIP SNAT at that exit point.

When a pod sends traffic to a CUDN IP (10.200.x.x), OVN recognizes the destination as belonging to the user-defined network. Instead of routing it externally, OVN hands it to FRR via RouteAdvertisements, which advertises it over BGP to the c8000v, and onward to the remote cluster's FRR and OVN. The traffic never reaches the external egress path where SNAT would be applied — it stays within the CUDN routing domain the entire way. The pod's CUDN IP is preserved as the source.

## Prerequisites

- Hub and spoke clusters deployed (OCP 4.20+)
- `ansible-playbook common/setup.yaml` completed

## Automated Deployment

```bash
# 1. Deploy CUDNs, BGP peering, and label nodes egress-assignable
ansible-playbook lab03-egressip/lab03.yaml

# 2. Run three-phase identity verification
ansible-playbook lab03-egressip/verify.yaml

# Individual phases
ansible-playbook lab03-egressip/verify.yaml -t phase1   # Default egress
ansible-playbook lab03-egressip/verify.yaml -t phase2   # EgressIP applied
ansible-playbook lab03-egressip/verify.yaml -t phase3   # Cross-cluster identity
```

## Files Reference

| File | Description |
|------|-------------|
| `lab03.yaml` | Deploy CUDNs, BGP peering, and label nodes egress-assignable |
| `verify.yaml` | Three-phase EgressIP and pod identity demonstration |
| `manifest/hub-egressip.yaml` | EgressIP CR (172.16.252.70 for cudn-hub-ns) |
| `manifest/hub-cudn.yaml` | Hub namespace + ClusterUserDefinedNetwork |
| `manifest/hub-bgp.yaml` | Hub FRRConfiguration + RouteAdvertisements |
| `manifest/spoke-cudn.yaml` | Spoke namespace + ClusterUserDefinedNetwork |
| `manifest/spoke-bgp.yaml` | Spoke FRRConfiguration + RouteAdvertisements |
