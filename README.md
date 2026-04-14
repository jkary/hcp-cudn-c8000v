# BGP-Routed CUDNs Between Hub and Spoke Clusters via c8000v

A series of labs demonstrating cross-cluster pod networking on OpenShift using ClusterUserDefinedNetworks (CUDNs), BGP via FRR-K8s, and Cisco c8000v virtual routers in a dual-spine topology.

## Network Design

```
                 lab-network (172.16.252.0/24)
                 +-----------------+
                 |    c8000v-1     |
                 |  AS 64514       |
                 |  172.16.252.50  |
                 +--------+--------+
                          |
   .61     .62     .63    |    .64
  +----+ +----+ +----+   |   +----+
  | M1 | | M2 | | M3 |---+---| W1 |
  +----+ +----+ +----+       +----+
    |      |      |  iBGP       |
  +----+ +----+ +----+  |    +----+
  | M1 | | M2 | | M3 |--+----| W1 |
  +----+ +----+ +----+  |    +----+
   .61     .62     .63   |    .64
                         |
                 +-------+--------+
                 |    c8000v-2     |
                 |  AS 64514       |
                 |  172.16.253.50  |
                 +-----------------+
                 lab-network-253 (172.16.253.0/24)

  Hub AS 64512                  Spoke AS 64513
  CUDN: 10.100.0.0/16          CUDN: 10.200.0.0/16
```

Each cluster node is dual-homed to both c8000v routers. The two routers share AS 64514 and peer via iBGP, providing ECMP for cross-cluster traffic.

## The Stack

| Layer | Purpose |
|-------|---------|
| CUDN | Defines the pod network |
| Supernet | Logical address allocation (/16) |
| Per-node subnet | Physical distribution (/24 per node) |
| FRR / BGP | Advertises reachability across clusters |
| iBGP | Dual-spine route exchange between c8000v routers |
| VRF | Isolates networks (multi-tenant) |
| ECMP / LB | Distributes traffic across multiple paths |

## Labs

| Lab | Topic | Depends On |
|-----|-------|------------|
| [Lab 01](lab01-cudn-bgp/lab01.md) | CUDN supernets and BGP advertising options | common |
| [Lab 02](lab02-flow-tracing/lab02.md) | Flow traceability using tshark | Lab 01 |
| [Lab 03](lab03-egressip/lab03.md) | EgressIP and dual-spine ECMP | common |
| [Lab 04](lab04-vrf-lite/lab04.md) | VRF lite and multi-tenant isolation | common |

## Prerequisites

- Hub and spoke OpenShift clusters (OCP 4.20+) running as KVM VMs
- c8000v qcow2 image at `/home/jkary/c8000v-17.06.03/virtioa.qcow2`
- Both clusters share the `lab-network` (172.16.252.0/24) L2 network
- Ansible with `ansible.netcommon`, `cisco.ios`, and `kubernetes.core` collections
- `genisoimage` for day0 ISO creation

## Quick Start

```bash
# 1. Edit site-specific variables (IPs, ASNs, kubeconfig paths)
vi common/manifest/vars.yaml

# 2. Run common setup (creates c8000v VMs, attaches second NIC, configures BGP + FRR)
ap common/setup.yaml

# 3. Run a lab
ap lab01-cudn-bgp/lab01.yaml
ap lab01-cudn-bgp/verify.yaml

# Teardown between labs
ap common/teardown.yaml
```

The common setup is idempotent — it skips VM creation if they already exist, and skips NIC attachment if already present.

## Customizing for Your Environment

All site-specific values are in `common/manifest/vars.yaml`:
- c8000v router IPs, ASN, and credentials
- Hub and spoke kubeconfig paths
- Node IPs on both L2 segments (252.x and 253.x)
- CUDN CIDRs and subnet sizes
- Libvirt disk paths and cluster VM names/MACs

Each lab's K8s manifests are also available as kustomize overlays:
```bash
oc apply -k lab01-cudn-bgp/manifest/
```
