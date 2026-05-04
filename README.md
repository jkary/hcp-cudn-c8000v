# BGP-Routed CUDNs Between Hub and Spoke Clusters via c8000v

A series of labs demonstrating cross-cluster pod networking on OpenShift (OCP 4.20+) using ClusterUserDefinedNetworks (CUDNs), BGP via FRR-K8s, and a Cisco c8000v virtual router as the BGP relay.

## Hosted Control Planes

In a traditional OpenShift cluster, the control plane (API server, etcd, controllers) runs on dedicated master nodes within the same cluster. With Hosted Control Planes (HCP), the control plane for one cluster runs as pods inside a different "management" cluster. This decouples the control plane from the worker nodes that run user workloads.

In this lab environment, the **hub cluster** (M1–M3) serves as both control plane and worker nodes for the hub, and also hosts the control plane for the **spoke cluster**. The spoke cluster's only infrastructure is a single worker node (W1) — its control plane runs as pods on the hub. Despite sharing the same physical network, the hub and spoke are separate OpenShift clusters with independent pod networks, which is why BGP routing is needed to connect them.

## Network Design

```
  Hub Cluster (AS 64512)              c8000v (AS 64514)             Spoke (AS 64513)
  CUDN: 10.100.0.0/16                                              CUDN: 10.200.0.0/16

  +------+  +------+  +------+     +-------------------+           +------+
  |  M1  |  |  M2  |  |  M3  |     |      c8000v       |           |  W1  |
  |  .61 |  |  .62 |  |  .63 |     | Gi1: 172.16.252.50|           |  .64 |
  +--+---+  +--+---+  +--+---+     | Gi2: 172.16.253.50|           +--+---+
     |         |         |          +--+-------------+--+              |
     |         |         |             |             |                 |
  ---+---------+---------+-------------+---       ---+-----------------+---
       lab-network (172.16.252.0/24)                lab-network-253 (172.16.253.0/24)
```

The c8000v has two interfaces — one on each cluster's network — and routes CUDN traffic between them. Hub nodes peer with c8000v on Gi1 (172.16.252.50), spoke peers on Gi2 (172.16.253.50).

## The Stack

| Layer           | Purpose                                          |
| --------------- | ------------------------------------------------ |
| CUDN            | Defines the pod network                          |
| Supernet        | Logical address allocation (/16)                 |
| Per-node subnet | Physical distribution (/24 per node)             |
| FRR / BGP       | Advertises reachability across clusters          |
| VRF             | Isolates networks (multi-tenant)                 |

## Labs

| Lab                                   | Topic                                      | Depends On |
| ------------------------------------- | ------------------------------------------ | ---------- |
| [Lab 01](lab01-cudn-bgp/lab01.md)     | CUDN supernets and BGP advertising options | common     |
| [Lab 02](lab02-flow-tracing/lab02.md) | Flow traceability using tshark             | Lab 01     |
| [Lab 03](lab03-egressip/lab03.md)     | EgressIP and pod identity                  | common     |
| [Lab 04](lab04-vrf-lite/lab04.md)     | VRF lite and multi-tenant isolation        | common     |

## Prerequisites

- Hub and spoke OpenShift clusters (OCP 4.20+) running as KVM VMs
- c8000v qcow2 image (set `c8000v_source_qcow2` in `common/manifest/vars.yaml`)
- Two libvirt networks: `lab-network` (172.16.252.0/24) for the hub cluster and c8000v Gi1, `lab-network-253` (172.16.253.0/24) for the spoke cluster and c8000v Gi2
- Ansible with `ansible.netcommon`, `cisco.ios`, and `kubernetes.core` collections
- `genisoimage` for day0 ISO creation

## Quick Start

```bash
# 1. Edit site-specific variables (IPs, ASNs, kubeconfig paths)
vi common/manifest/vars.yaml

# 2. Run common setup (creates c8000v VM, attaches second NIC, configures BGP + FRR)
ansible-playbook common/setup.yaml

# 3. Run a lab
ansible-playbook lab01-cudn-bgp/lab01.yaml
ansible-playbook lab01-cudn-bgp/verify.yaml

# Teardown between labs
ansible-playbook common/teardown.yaml
```

The common setup is idempotent — it skips VM creation if VMs already exist, and skips NIC attachment if already present.

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
