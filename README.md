# BGP-Routed CUDNs Between Hub and Spoke Clusters via c8000v

A series of labs demonstrating cross-cluster pod networking on OpenShift using ClusterUserDefinedNetworks (CUDNs), BGP via FRR-K8s, and a Cisco c8000v virtual router.

## Network Design

```
                  +-----------------+
                  |    c8000v       |
                  |  AS 64514       |
                  |  172.16.252.50  |
                  +--------+--------+
                           |
            +--------------+--------------+
            |         L2 Network          |
            |      172.16.252.0/24        |
            +--+------+------+------+----+
               |      |      |      |
            .61|   .62|   .63|   .64|
          +----+----+ |      | +----+----+
          | master-1| |      | | worker-1|
          +---------+ |      | +---------+
          +----+----+ |      |   Spoke
          | master-2|-+      |   AS 64513
          +---------+        |   CUDN: 10.200.0.0/16
          +----+----+        |
          | master-3|--------+
          +---------+
            Hub
            AS 64512
            CUDN: 10.100.0.0/16
```

## The Stack

| Layer | Purpose |
|-------|---------|
| CUDN | Defines the pod network |
| Supernet | Logical address allocation (/16) |
| Per-node subnet | Physical distribution (/24 per node) |
| FRR / BGP | Advertises reachability across clusters |
| VRF | Isolates networks (multi-tenant) |
| ECMP / LB | Distributes traffic |

## Labs

| Lab | Topic | Depends On |
|-----|-------|------------|
| [Lab 01](lab01-cudn-bgp/lab01.md) | CUDN supernets and BGP advertising options | common |
| [Lab 02](lab02-flow-tracing/lab02.md) | Flow traceability using tshark | Lab 01 |
| [Lab 03](lab03-egressip/lab03.md) | EgressIP and load balancing | common |
| [Lab 04](lab04-vrf-lite/lab04.md) | VRF lite and multi-tenant isolation | common |

## Prerequisites

- Hub and spoke OpenShift clusters (OCP 4.20+)
- c8000v router running on KVM with an interface on the shared L2 network
- Both clusters share the same L2 network
- Ansible with `ansible.netcommon`, `cisco.ios`, and `kubernetes.core` collections

## Quick Start

```bash
# 1. Edit site-specific variables (IPs, ASNs, kubeconfig paths)
vi common/manifest/vars.yaml

# 2. Run common setup (c8000v base BGP + cluster FRR enablement)
ap common/setup.yaml

# 3. Run a lab
ap lab01-cudn-bgp/lab01.yaml
ap lab01-cudn-bgp/verify.yaml

# Teardown between labs
ap common/teardown.yaml
```

## Customizing for Your Environment

All site-specific values are in `common/manifest/vars.yaml`:
- c8000v router IP, ASN, and credentials
- Hub and spoke kubeconfig paths
- Node IPs and ASNs
- CUDN CIDRs and subnet sizes

Each lab's K8s manifests are also available as kustomize overlays:
```bash
oc apply -k lab01-cudn-bgp/manifest/
```
