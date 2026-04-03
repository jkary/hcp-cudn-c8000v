# Lab 04: VRF Lite and Multi-Tenant Isolation

**Status:** Planned — not yet implemented.

## Overview

This lab creates a VRF-based network isolation setup where multiple tenants share the same physical infrastructure and BGP relay but have fully isolated routing domains. It shows how OpenShift handles the Linux VRF pieces and how c8000v separates traffic using per-VRF address families.

## What You'll Learn

- Configuring VRF definitions on c8000v (`vrf definition`, `rd`, `route-target`)
- Per-VRF BGP address families on the router
- Creating isolated CUDNs per tenant with non-overlapping CIDRs
- How Linux VRF tables appear on OCP nodes
- Proving tenant isolation: tenant A pods cannot reach tenant B pods
- Inspecting VRF routing tables (`show ip route vrf`, `ip vrf show` on nodes)

## Depends On

- `common/setup.yaml`
- Note: This lab replaces the c8000v BGP config (VRF restructures the BGP process)
