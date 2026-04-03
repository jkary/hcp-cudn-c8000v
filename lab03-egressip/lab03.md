# Lab 03: EgressIP and ECMP Load Balancing

## Overview

This lab demonstrates that CUDNs alone don't solve egress identity. Without EgressIP, pods egress via SNAT to the node IP -- unpredictable in multi-node clusters. This lab applies EgressIP to pin a stable source IP for outbound traffic, and configures ECMP on the c8000v to distribute return traffic across all hub nodes.

## What You'll Learn

- Default CUDN egress behavior (SNAT to node IP)
- Applying EgressIP CRs to pin egress source addresses
- How EgressIP interacts with CUDNs (external egress only, not cross-cluster)
- Configuring ECMP via `maximum-paths` on the c8000v
- Verifying load distribution across BGP paths using CEF

## Network Design

```
                  +-----------------+
                  |    c8000v       |
                  |  AS 64514       |
                  |  172.16.252.50  |
                  |  ECMP: 3 paths  |
                  +--------+--------+
                           |
            +--------------+--------------+
            |         L2 Network          |
            |      172.16.252.0/24        |
            +--+------+------+------+----+
               |      |      |      |
            .61|   .62|   .63|   .64|     .70 (EgressIP)
          +----+----+ |      | +----+----+
          | master-1| |      | | worker-1|
          +---------+ |      | +---------+
          +----+----+ |      |   Spoke
          | master-2|-+      |   AS 64513
          +---------+        |
          +----+----+        |
          | master-3|--------+
          +---------+
            Hub AS 64512
            CUDN: 10.100.0.0/16
            EgressIP: 172.16.252.70
```

## Prerequisites

- Hub and spoke clusters deployed (OCP 4.20+)
- `ansible-playbook common/setup.yaml` completed

## Automated Deployment

```bash
# 1. Deploy infrastructure (CUDNs, BGP, ECMP on c8000v)
ansible-playbook lab03-egressip/lab03.yaml

# 2. Run three-phase verification
ansible-playbook lab03-egressip/verify.yaml

# Individual phases
ansible-playbook lab03-egressip/verify.yaml -t phase1   # Default egress
ansible-playbook lab03-egressip/verify.yaml -t phase2   # EgressIP applied
ansible-playbook lab03-egressip/verify.yaml -t phase3   # ECMP verification
```

## Key Learnings

### EgressIP only affects external egress

EgressIP SNATs traffic leaving the cluster to external destinations. Cross-cluster CUDN traffic (hub pod -> spoke pod via BGP) is NOT affected -- it uses the pod's CUDN IP as the source, routed through the BGP path.

### ECMP distributes return traffic

With `maximum-paths 3` on the c8000v, return traffic to 10.100.0.0/16 is distributed across all 3 hub nodes using per-destination hashing. This is orthogonal to EgressIP: outbound uses a deterministic EgressIP, inbound is load-balanced.

### EgressIP assignment

When no `nodeSelector` is specified on the EgressIP CR, OVN automatically assigns it to a healthy node. The assigned node hosts the EgressIP as a secondary address and handles SNAT for all matching pods.

## Files Reference

| File | Description |
|------|-------------|
| `lab03.yaml` | Deploy CUDNs, BGP peering, and ECMP config |
| `verify.yaml` | Three-phase EgressIP and ECMP demonstration |
| `manifest/hub-egressip.yaml` | EgressIP CR (172.16.252.70 for cudn-hub-ns) |
| `manifest/c8000v-ecmp.cfg` | c8000v ECMP config (maximum-paths 3) |
| `manifest/hub-cudn.yaml` | Hub namespace + ClusterUserDefinedNetwork |
| `manifest/hub-bgp.yaml` | Hub FRRConfiguration + RouteAdvertisements |
| `manifest/spoke-cudn.yaml` | Spoke namespace + ClusterUserDefinedNetwork |
| `manifest/spoke-bgp.yaml` | Spoke FRRConfiguration + RouteAdvertisements |
