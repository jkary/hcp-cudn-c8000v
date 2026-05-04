# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repo Is

A series of labs demonstrating cross-cluster pod networking on OpenShift (OCP 4.20+) using ClusterUserDefinedNetworks (CUDNs), BGP via FRR-K8s, and a Cisco c8000v virtual router as the BGP relay.

## Architecture

The **hub cluster** (M1–M3) serves as both its own control plane and hosts the control plane for the **spoke cluster** via Hosted Control Planes (HCP). The spoke's only infrastructure is a single worker node (W1). Despite sharing the same physical network, hub and spoke are separate OpenShift clusters with independent pod networks — BGP routing connects them.

Three BGP autonomous systems:

- **Hub cluster** (AS 64512): 3 master nodes on 172.16.252.0/24, CUDN 10.100.0.0/16
- **Spoke cluster** (AS 64513): 1 worker node on 172.16.253.0/24, CUDN 10.200.0.0/16
- **c8000v** (AS 64514): BGP relay with Gi1 on 172.16.252.0/24 and Gi2 on 172.16.253.0/24

The stack: OVN <-> RouteAdvertisements <-> FRR-K8s <-> c8000v (BGP relay).

The c8000v has two interfaces — one on each cluster's network — and routes CUDN traffic between them. Hub nodes peer with c8000v on Gi1 (172.16.252.50), spoke peers on Gi2 (172.16.253.50).

## Lab Dependencies

All labs require `common/setup.yaml` first. Lab 02 (flow tracing) additionally requires Lab 01. Labs 03 and 04 are independent of each other.

## Key Commands

```bash
# Common setup (required before any lab)
ansible-playbook common/setup.yaml              # Full setup: infra + c8000v BGP + cluster FRR
ansible-playbook common/setup.yaml -t infra     # Libvirt network, VM, spoke second NIC only
ansible-playbook common/setup.yaml -t c8000v    # c8000v BGP config only
ansible-playbook common/setup.yaml -t hub       # Hub cluster only
ansible-playbook common/setup.yaml -t spoke     # Spoke cluster only

# Run a lab
ansible-playbook lab01-cudn-bgp/lab01.yaml
ansible-playbook lab01-cudn-bgp/verify.yaml

# Apply manifests via kustomize
oc apply -k lab01-cudn-bgp/manifest/

# Teardown between labs
ansible-playbook common/teardown.yaml
ansible-playbook common/teardown.yaml -t infra  # Remove spoke second NIC, lab-network-253
```

The common setup is idempotent — it skips VM creation if VMs already exist and skips NIC attachment if already present.

## Required Ansible Collections

`ansible.netcommon`, `cisco.ios`, `kubernetes.core`

## Design Patterns

- **All site-specific values in `common/manifest/vars.yaml`** — IPs, ASNs, CIDRs, kubeconfig paths. Playbooks load this via `vars_files`.
- **Pre-check + fail fast** — lab playbooks verify FRR pods are running before proceeding. If not: "Run `ansible-playbook common/setup.yaml` first."
- **Kustomize: common base + per-lab overlays** — each lab's `manifest/kustomization.yaml` references `../../common/manifest` and adds lab-specific resources.
- **VM vs cluster separation** — c8000v config (`.cfg`/`.cfg.j2` files) is IOS syntax pushed via `ansible.netcommon.cli_config`. K8s manifests (`.yaml`) are applied via `kubernetes.core.k8s`.
- **c8000v config is a Jinja2 template** (`c8000v-bgp-base.cfg.j2`) rendered at runtime from vars. Lab-specific IOS configs are additive snippets except lab04 (VRF) which replaces the entire BGP config.
- **Inventory** — single `inventory.yml` with `localhost` and `c8000v` hosts. c8000v connection uses `paramiko` SSH (`ansible_network_cli_ssh_type: paramiko`).
- **Verify playbooks** — each lab has a `verify.yaml` that checks the lab's expected state (BGP sessions, routes, connectivity). Run after the lab playbook to confirm success.
- **VM provisioning** — `common/setup.yaml` creates the c8000v VM from a qcow2 template (`/home/jkary/c8000v-17.06.03/virtioa.qcow2`) using a backing-file copy. Day0 bootstrap config is rendered from `c8000v-day0.cfg.j2` and packaged as an ISO via `genisoimage`. The file must be named `iosxe_config.txt` inside the ISO.
- **Spoke second NIC** — the spoke worker VM gets a second NIC on `lab-network-253` via `virsh attach-device` with explicit PCI address (bus 0x06) and fixed MAC. NMState NNCPs use `identifier: mac-address` to configure the interface.

## Lab-Specific Notes

**Lab03 (EgressIP)**: Demonstrates EgressIP for stable pod identity. Three verification phases: default egress (node IP as source), EgressIP applied (172.16.252.70 as source), and cross-cluster CUDN identity (pod CUDN IP preserved, EgressIP not applied). Includes tcpdump proof that cross-cluster traffic uses CUDN IPs.

**Lab04 (VRF Lite)**: Replaces the entire c8000v BGP config (not additive). Uses NMState for VLAN sub-interfaces and `nsenter -t 1 -n` from FRR pods to move VLAN interfaces into OVN-created VRF devices. Run `common/teardown.yaml` + `common/setup.yaml` to restore base config after this lab.

## Critical Gotchas

- **Two Network operator settings required**: `additionalRoutingCapabilities.providers: [FRR]` AND `routeAdvertisements: Enabled`. Without both, BGP sessions work but pods can't use the routes.
- **`disableMP: true` is mandatory** on FRRConfiguration BGP neighbors when using RouteAdvertisements.
- **RouteAdvertisements API (OCP 4.20)**: `advertisements` is a string array (`- PodNetwork`), `networkSelectors` is plural, `nodeSelector: {}` is required.
- **Pod IPs**: `oc get pod -o wide` shows the default network IP. Use `oc exec <pod> -- ip -4 addr show ovn-udn1` for the CUDN IP.
- **HCP spoke caveat**: HostedCluster controller may reconcile away Network operator patches.
- **c8000v day0 ISO filename**: The file inside the ISO must be named `iosxe_config.txt` exactly. Use `genisoimage -r` (Rock Ridge) to preserve the full filename.
- **PCI slot exhaustion**: Cluster VMs using PCIe topology may have all root ports occupied. Use `virsh attach-device` with explicit PCI address on the pcie-to-pci-bridge (bus 0x06) instead of `virsh attach-interface`.
- **NIC naming on hot-add**: Hot-added NICs get unpredictable kernel names. Use fixed MAC addresses and NMState `identifier: mac-address` in NNCPs to match reliably.
