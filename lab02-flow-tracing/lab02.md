# Lab 02: Flow Traceability Using tshark

**Status:** Planned — not yet implemented.

## Overview

This lab demonstrates how to trace packet flows across the CUDN/BGP path using tshark. It deploys privileged capture pods on each node, generates identifiable traffic between hub and spoke pods, and collects PCAPs showing the packet at each hop.

## What You'll Learn

- How to capture on the `ovn-udn1` interface inside pods
- How packets traverse: pod → OVN → node routing table → BGP next-hop → c8000v → destination node → destination pod
- Using tshark filters to isolate CUDN traffic from default cluster traffic
- Reading OVN Geneve encapsulation vs bare IP at each stage

## Depends On

- `common/setup.yaml`
- Lab 01 deployed (CUDNs and BGP peering active)
