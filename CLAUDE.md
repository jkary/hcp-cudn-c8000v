# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repo Is

A series of labs demonstrating cross-cluster pod networking on OpenShift (OCP 4.20+) using ClusterUserDefinedNetworks (CUDNs), BGP via FRR-K8s, and a Cisco c8000v virtual router as the BGP relay.

## Architecture

Three BGP autonomous systems with a dual-spine router topology:

- **Hub cluster** (AS 64512): 3 master nodes, CUDN 10.100.0.0/16, dual-homed to both spines
- **Spoke cluster** (AS 64513): 1 worker node, CUDN 10.200.0.0/16, dual-homed to both spines
- **c8000v-1** (AS 64514): BGP relay on 172.16.252.0/24 (`lab-network`)
- **c8000v-2** (AS 64514): BGP relay on 172.16.253.0/24 (`lab-network-253`), iBGP with c8000v-1

The stack: OVN <-> RouteAdvertisements <-> FRR-K8s <-> c8000v (BGP relay).

The two c8000v routers share AS 64514 and peer via iBGP on `lab-network` (172.16.252.50 <-> 172.16.252.51) with `next-hop-self`. This provides ECMP for cross-cluster traffic.

## Lab Dependencies

All labs require `common/setup.yaml` first. Lab 02 (flow tracing) additionally requires Lab 01. Labs 03 and 04 are independent of each other.

## Key Commands

`ap` is a shell alias for `ansible-playbook`.

```bash
# Common setup (required before any lab)
ap common/setup.yaml              # Full setup: infra + c8000v BGP + cluster FRR
ap common/setup.yaml -t infra     # Libvirt network, VMs, second NICs only
ap common/setup.yaml -t c8000v    # c8000v BGP config only
ap common/setup.yaml -t hub       # Hub cluster only
ap common/setup.yaml -t spoke     # Spoke cluster only

# Run a lab
ap lab01-cudn-bgp/lab01.yaml
ap lab01-cudn-bgp/verify.yaml

# Apply manifests via kustomize
oc apply -k lab01-cudn-bgp/manifest/

# Teardown between labs
ap common/teardown.yaml
ap common/teardown.yaml -t infra  # Remove c8000v-2, second NICs, lab-network-253
```

## Required Ansible Collections

`ansible.netcommon`, `cisco.ios`, `kubernetes.core`

## Design Patterns

- **All site-specific values in `common/manifest/vars.yaml`** — IPs, ASNs, CIDRs, kubeconfig paths. Playbooks load this via `vars_files`.
- **Pre-check + fail fast** — lab playbooks verify FRR pods are running before proceeding. If not: "Run `ap common/setup.yaml` first."
- **Kustomize: common base + per-lab overlays** — each lab's `manifest/kustomization.yaml` references `../../common/manifest` and adds lab-specific resources.
- **VM vs cluster separation** — c8000v config (`.cfg`/`.cfg.j2` files) is IOS syntax pushed via `ansible.netcommon.cli_config`. K8s manifests (`.yaml`) are applied via `kubernetes.core.k8s`.
- **c8000v config is a Jinja2 template** (`c8000v-bgp-base.cfg.j2`) rendered at runtime from vars. Lab-specific IOS configs are additive snippets except lab04 (VRF) which replaces the entire BGP config.
- **Inventory** — single `inventory.yml` with `localhost`, `c8000v`, and `c8000v-2` hosts. Both c8000v connections use `paramiko` SSH (`ansible_network_cli_ssh_type: paramiko`).
- **Verify playbooks** — each lab has a `verify.yaml` that checks the lab's expected state (BGP sessions, routes, connectivity). Run after the lab playbook to confirm success.
- **VM provisioning** — `common/setup.yaml` creates both c8000v VMs from a shared qcow2 template (`/home/jkary/c8000v-17.06.03/virtioa.qcow2`) using backing-file copies. Day0 bootstrap configs are rendered from `c8000v-day0.cfg.j2` and packaged as ISOs via `genisoimage`. The file must be named `iosxe_config.txt` inside the ISO.
- **Second NIC attachment** — cluster VMs get a second NIC on `lab-network-253` via `virsh attach-device` with explicit PCI address (bus 0x06) and fixed MAC. NMState NNCPs use `identifier: mac-address` to configure the interface.

## Lab-Specific Notes

**Lab03 (EgressIP + Dual-Spine ECMP)**: Installs NMState on both clusters, applies NNCPs for second NIC IPs, configures iBGP between c8000v-1 and c8000v-2 with `next-hop-self` and `maximum-paths ibgp 2`. FRRConfigurations get two BGP neighbors (one per spine).

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
