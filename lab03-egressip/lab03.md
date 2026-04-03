# Lab 03: EgressIP and Load Balancing

**Status:** Planned — not yet implemented.

## Overview

This lab demonstrates that CUDNs alone don't solve egress identity or traffic distribution. It configures EgressIP alongside CUDNs to provide stable source IPs for outbound traffic, and enables ECMP multipath to distribute traffic across multiple nodes.

## What You'll Learn

- Default CUDN egress behavior (SNAT to node IP — unpredictable in multi-node clusters)
- Applying EgressIP CRs to pin egress source addresses
- How EgressIP interacts with the CUDN primary interface vs the default cluster network
- Configuring ECMP in FRRConfiguration and c8000v (`maximum-paths`)
- Verifying load distribution across BGP paths

## Depends On

- `common/setup.yaml`
