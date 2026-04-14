# Lab 02: Flow Tracing with tshark

## Overview

Trace cross-cluster ICMP packet flows at each hop using tshark. Deploys capture pods with `NET_RAW` capability on both clusters, generates identifiable traffic between hub and spoke CUDN pods, and collects captures showing the packet at source and destination.

## What You'll Learn

- How to capture on the `ovn-udn1` interface inside CUDN pods
- How packets traverse: pod -> OVN -> node routing table -> BGP next-hop -> c8000v -> destination node -> destination pod
- Using tshark field filters to isolate CUDN traffic from default cluster traffic
- Correlating captures across clusters to trace end-to-end flow

## Depends On

- `common/setup.yaml`
- Lab 01 deployed (CUDNs, BGP peering, and test pods active)

## Architecture

```
Hub capture pod          c8000v-1 (BGP relay)      Spoke capture pod
10.100.x.x/24           172.16.252.50              10.200.0.x/24
    |                        |                          |
  ovn-udn1               GigabitEthernet1            ovn-udn1
    |                        |                          |
  [tshark]  ----ICMP---->  [routes]  ----ICMP---->  [tshark]
  capture                  BGP table                 capture
```

Traffic flows through c8000v-1 on `lab-network`. The capture pods are identical to lab01 test pods but with `NET_RAW` capability added, allowing tshark/tcpdump to capture on the CUDN interface.

## Automated Deployment

```bash
# Deploy capture pods (requires lab01 test pods running)
ansible-playbook lab02-flow-tracing/lab02.yaml

# Run flow traces and display results
ansible-playbook lab02-flow-tracing/verify.yaml
```

## Manual Steps

### Deploy capture pods

```bash
# Hub
export KUBECONFIG=/home/jkary/labs/hcp/deploy/auth/kubeconfig
oc apply -f lab02-flow-tracing/manifest/hub-capture.yaml

# Spoke
export KUBECONFIG=/home/jkary/labs/hcp/spoke/auth/kubeconfig
oc apply -f lab02-flow-tracing/manifest/spoke-capture.yaml
```

### Get CUDN IPs

```bash
# Hub capture pod
export KUBECONFIG=/home/jkary/labs/hcp/deploy/auth/kubeconfig
oc exec -n cudn-hub-ns hub-capture -- ip -4 addr show ovn-udn1

# Spoke capture pod
export KUBECONFIG=/home/jkary/labs/hcp/spoke/auth/kubeconfig
oc exec -n cudn-spoke-ns spoke-capture -- ip -4 addr show ovn-udn1
```

### Capture and trace

Start tshark on both pods, then generate traffic:

```bash
# Terminal 1: Capture on hub pod
export KUBECONFIG=/home/jkary/labs/hcp/deploy/auth/kubeconfig
oc exec -n cudn-hub-ns hub-capture -- tshark -i ovn-udn1 -f "icmp" \
  -T fields -e frame.number -e frame.time_relative \
  -e ip.src -e ip.dst -e icmp.type -e icmp.seq

# Terminal 2: Capture on spoke pod
export KUBECONFIG=/home/jkary/labs/hcp/spoke/auth/kubeconfig
oc exec -n cudn-spoke-ns spoke-capture -- tshark -i ovn-udn1 -f "icmp" \
  -T fields -e frame.number -e frame.time_relative \
  -e ip.src -e ip.dst -e icmp.type -e icmp.seq

# Terminal 3: Generate traffic
export KUBECONFIG=/home/jkary/labs/hcp/deploy/auth/kubeconfig
oc exec -n cudn-hub-ns hub-capture -- ping -c 5 <spoke-cudn-ip>
```

### Check c8000v perspective

```bash
ssh admin@172.16.252.50
show ip route bgp        # BGP-learned routes for CUDN prefixes
show ip bgp              # Full BGP table with next-hops
show interfaces GigabitEthernet1 | include packets   # Packet counters
```

## Expected Output

The tshark captures on both pods should show matching ICMP echo request/reply pairs:

- **Hub pod capture**: outgoing echo requests (type 8) to spoke IP, incoming echo replies (type 0) from spoke IP
- **Spoke pod capture**: incoming echo requests (type 8) from hub IP, outgoing echo replies (type 0) to hub IP
- **Sequence numbers match** across both captures, confirming end-to-end flow

## Files Reference

| File | Description |
|------|-------------|
| `lab02.yaml` | Deploy capture pods with NET_RAW capability |
| `verify.yaml` | Run tshark captures, generate traffic, display flow traces |
| `manifest/hub-capture.yaml` | Hub capture pod (NET_RAW enabled) |
| `manifest/spoke-capture.yaml` | Spoke capture pod (NET_RAW enabled) |
