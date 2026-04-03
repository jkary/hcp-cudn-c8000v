# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repo Is

A series of labs demonstrating cross-cluster pod networking on OpenShift (OCP 4.20+) using ClusterUserDefinedNetworks (CUDNs), BGP via FRR-K8s, and a Cisco c8000v virtual router as the BGP relay.

## Architecture

Three BGP autonomous systems connected via eBGP on a shared L2 network (172.16.252.0/24):

- **Hub cluster** (AS 64512): 3 master nodes, CUDN 10.100.0.0/16
- **Spoke cluster** (AS 64513): 1 worker node, CUDN 10.200.0.0/16
- **c8000v router** (AS 64514): BGP relay between hub and spoke

The stack: OVN <-> RouteAdvertisements <-> FRR-K8s <-> c8000v (BGP relay).

## Repo Structure

```
common/                    # Shared infrastructure (run first)
  setup.yaml               # c8000v base BGP + cluster FRR/RouteAdv enablement
  teardown.yaml            # Reset between labs
  manifest/
    vars.yaml              # ALL site-specific values (edit this for your environment)
    c8000v-bgp-base.cfg.j2 # Jinja2 template for c8000v BGP config
    kustomization.yaml      # Kustomize base

lab01-cudn-bgp/            # CUDN supernets and BGP advertising
lab02-flow-tracing/        # tshark flow traceability (planned)
lab03-egressip/            # EgressIP and load balancing (planned)
lab04-vrf-lite/            # VRF lite multi-tenant isolation (planned)
```

Each lab has: `labNN.yaml` (playbook), `verify.yaml`, `manifest/` (kustomize overlay + K8s manifests).

## Key Commands

```bash
# Common setup (required before any lab)
ap common/setup.yaml
ap common/setup.yaml -t c8000v    # c8000v only
ap common/setup.yaml -t hub       # Hub cluster only
ap common/setup.yaml -t spoke     # Spoke cluster only

# Run a lab
ap lab01-cudn-bgp/lab01.yaml
ap lab01-cudn-bgp/verify.yaml

# Apply manifests via kustomize
oc apply -k lab01-cudn-bgp/manifest/

# Teardown
ap common/teardown.yaml
```

## Design Patterns

- **All site-specific values in `common/manifest/vars.yaml`** — IPs, ASNs, CIDRs, kubeconfig paths. Playbooks load this via `vars_files`.
- **Pre-check + fail fast** — lab playbooks verify FRR pods are running before proceeding. If not: "Run `ap common/setup.yaml` first."
- **Kustomize: common base + per-lab overlays** — each lab's `manifest/kustomization.yaml` references `../../common/manifest` and adds lab-specific resources.
- **VM vs cluster separation** — c8000v config (`.cfg`/`.cfg.j2` files) is IOS syntax pushed via `ansible.netcommon.cli_config`. K8s manifests (`.yaml`) are applied via `kubernetes.core.k8s`.
- **c8000v config is a Jinja2 template** (`c8000v-bgp-base.cfg.j2`) rendered at runtime from vars. Lab-specific IOS configs are additive snippets except lab04 (VRF) which replaces the entire BGP config.

## Critical Gotchas

- **Two Network operator settings required**: `additionalRoutingCapabilities.providers: [FRR]` AND `routeAdvertisements: Enabled`. Without both, BGP sessions work but pods can't use the routes.
- **`disableMP: true` is mandatory** on FRRConfiguration BGP neighbors when using RouteAdvertisements.
- **RouteAdvertisements API (OCP 4.20)**: `advertisements` is a string array (`- PodNetwork`), `networkSelectors` is plural, `nodeSelector: {}` is required.
- **Pod IPs**: `oc get pod -o wide` shows the default network IP. Use `oc exec <pod> -- ip -4 addr show ovn-udn1` for the CUDN IP.
- **HCP spoke caveat**: HostedCluster controller may reconcile away Network operator patches.
