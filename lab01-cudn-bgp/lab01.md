# BGP-Routed CUDNs Between Hub and Spoke Clusters via c8000v

This guide sets up cross-cluster pod networking between a hub (management) and spoke (HCP hosted) OpenShift cluster using ClusterUserDefinedNetworks (CUDNs), BGP via FRR-K8s, and a Cisco c8000v router as the BGP relay.

## Network Design

This lab uses c8000v-1 (on `lab-network`) for BGP peering. The dual-spine topology (c8000v-2 on `lab-network-253`) is created by `common/setup.yaml` but used in lab03.

```
                  +-----------------+
                  |    c8000v-1     |
                  |  AS 64514       |
                  |  172.16.252.50  |
                  +--------+--------+
                           |
            +--------------+--------------+
            |     lab-network             |
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

| Component | ASN | CUDN CIDR | Node IPs |
|-----------|-----|-----------|----------|
| Hub cluster | 64512 | 10.100.0.0/16 (/24 per node) | 172.16.252.61-63 |
| Spoke cluster | 64513 | 10.200.0.0/16 (/24 per node) | 172.16.252.64 |
| c8000v-1 | 64514 | N/A | 172.16.252.50 |
| c8000v-2 | 64514 | N/A | 172.16.253.50 (used in lab03) |

## Prerequisites

- Hub and spoke clusters deployed (OCP 4.20+)
- `ansible-playbook common/setup.yaml` completed (creates c8000v VMs and configures FRR)

## Step-by-Step Reproduction

### Phase 1: Configure c8000v BGP

SSH to the c8000v and apply the BGP configuration:

```bash
ssh admin@172.16.252.50
```

```
enable
configure terminal

router bgp 64514
 bgp router-id 172.16.252.50
 bgp log-neighbor-changes
 neighbor 172.16.252.61 remote-as 64512
 neighbor 172.16.252.61 description hub-master-0
 neighbor 172.16.252.62 remote-as 64512
 neighbor 172.16.252.62 description hub-master-1
 neighbor 172.16.252.63 remote-as 64512
 neighbor 172.16.252.63 description hub-master-2
 neighbor 172.16.252.64 remote-as 64513
 neighbor 172.16.252.64 description spoke-worker-1
 address-family ipv4 unicast
  neighbor 172.16.252.61 activate
  neighbor 172.16.252.62 activate
  neighbor 172.16.252.63 activate
  neighbor 172.16.252.64 activate
 exit-address-family
end

write memory
```

Verify:

```
show ip bgp summary
```

All neighbors will show `Idle` until the clusters have FRR-K8s configured.

### Phase 2: Hub Cluster - Enable FRR and Route Advertisements

Set KUBECONFIG to the hub cluster:

```bash
export KUBECONFIG=/home/jkary/labs/hcp/deploy/auth/kubeconfig
```

#### 2a. Patch the Network operator

This does two things:
- Enables FRR-K8s (deploys FRR pods on every node)
- Enables RouteAdvertisements (deploys the CRD and enables OVN/FRR integration)

```bash
oc patch Network.operator.openshift.io cluster --type=merge --patch '{
  "spec": {
    "additionalRoutingCapabilities": {
      "providers": ["FRR"]
    },
    "defaultNetwork": {
      "ovnKubernetesConfig": {
        "routeAdvertisements": "Enabled"
      }
    }
  }
}'
```

Wait for FRR-K8s pods to be ready:

```bash
oc get pods -n openshift-frr-k8s -w
```

Wait for the RouteAdvertisements CRD to appear:

```bash
oc get crd routeadvertisements.k8s.ovn.org
```

#### 2b. Create CUDN namespace and network

```bash
oc apply -f lab01-cudn-bgp/manifest/hub-cudn.yaml
```

This creates:
- Namespace `cudn-hub-ns` with label `k8s.ovn.org/primary-user-defined-network: ""`
- ClusterUserDefinedNetwork `hub-network` (Layer3, 10.100.0.0/16, /24 per node, Primary role)

#### 2c. Create FRRConfiguration and RouteAdvertisements

```bash
oc apply -f lab01-cudn-bgp/manifest/hub-bgp.yaml
```

This creates:
- **FRRConfiguration** `hub-bgp` in `openshift-frr-k8s` namespace — establishes eBGP session from AS 64512 to c8000v at AS 64514, advertises 10.100.0.0/16 prefix
- **RouteAdvertisements** `hub-cudn-routes` — tells OVN to integrate FRR-learned routes into the CUDN data plane, selecting CUDNs with label `advertise: "true"`

Verify BGP sessions:

```bash
oc get bgpsessionstates -n openshift-frr-k8s
```

All nodes should show `Established`.

### Phase 3: Spoke Cluster - Enable FRR and Route Advertisements

Set KUBECONFIG to the spoke cluster:

```bash
export KUBECONFIG=/home/jkary/labs/hcp/spoke/auth/kubeconfig
```

#### 3a. Patch the spoke Network operator

```bash
oc patch Network.operator.openshift.io cluster --type=merge --patch '{
  "spec": {
    "additionalRoutingCapabilities": {
      "providers": ["FRR"]
    },
    "defaultNetwork": {
      "ovnKubernetesConfig": {
        "routeAdvertisements": "Enabled"
      }
    }
  }
}'
```

Wait for FRR-K8s pods and RouteAdvertisements CRD as above.

> **Note:** For HCP-managed spoke clusters, the Network operator patch may get
> reconciled away by the HostedCluster controller. If that happens, the
> HostedCluster CR does not currently support `additionalRoutingCapabilities`
> in its networking spec. Monitor the spoke's Network operator to confirm the
> patch persists.

#### 3b. Create CUDN namespace, network, and BGP peering

```bash
oc apply -f lab01-cudn-bgp/manifest/spoke-cudn.yaml
oc apply -f lab01-cudn-bgp/manifest/spoke-bgp.yaml
```

Verify:

```bash
oc get bgpsessionstates -n openshift-frr-k8s
oc get routeadvertisements spoke-cudn-routes -o jsonpath='{.status}'
```

### Phase 4: Verification

#### Check c8000v BGP routes

```
show ip bgp summary    # all 4 neighbors should be Established
show ip bgp            # should show 10.100.x.0/24 and 10.200.0.0/24 routes
```

Expected output:

```
     Network          Next Hop            Metric LocPrf Weight Path
 *>   10.100.0.0/24    172.16.252.62            0             0 64512 i
 *>   10.100.0.0/16    172.16.252.61            0             0 64512 i
 *>   10.100.1.0/24    172.16.252.61            0             0 64512 i
 *>   10.100.2.0/24    172.16.252.63            0             0 64512 i
 *>   10.200.0.0/24    172.16.252.64            0             0 64513 i
 *>   10.200.0.0/16    172.16.252.64            0             0 64513 i
```

The /24 subnets are per-node allocations advertised by RouteAdvertisements. The /16 aggregates come from the FRRConfiguration `prefixes` field.

#### Deploy test pods

The `verify.yaml` playbook automatically deploys test pods on both clusters. To deploy them manually:

```bash
# Hub
export KUBECONFIG=/home/jkary/labs/hcp/deploy/auth/kubeconfig
oc apply -f lab01-cudn-bgp/manifest/test-pods.yaml

# Spoke
export KUBECONFIG=/home/jkary/labs/hcp/spoke/auth/kubeconfig
oc apply -f lab01-cudn-bgp/manifest/test-pods.yaml
```

> **Note:** The test-pods.yaml file contains pods for both clusters. Each pod
> targets a different namespace (`cudn-hub-ns` / `cudn-spoke-ns`), so applying
> the full file to each cluster is safe — only the matching namespace's pod will
> be created.

#### Get CUDN IPs

Pods have two interfaces: `eth0` (default cluster network, infrastructure-locked) and `ovn-udn1` (CUDN, primary). The CUDN IP is on `ovn-udn1`:

```bash
# Hub pod
export KUBECONFIG=/home/jkary/labs/hcp/deploy/auth/kubeconfig
oc exec -n cudn-hub-ns hub-test -- ip -4 addr show ovn-udn1
# Example: 10.100.0.8/24

# Spoke pod
export KUBECONFIG=/home/jkary/labs/hcp/spoke/auth/kubeconfig
oc exec -n cudn-spoke-ns spoke-test -- ip -4 addr show ovn-udn1
# Example: 10.200.0.9/24
```

> **Note:** `oc get pod -o wide` shows the default network IP (10.132.x.x /
> 10.133.x.x), not the CUDN IP. Use `ip addr` inside the pod or check the
> `k8s.ovn.org/pod-networks` annotation.

#### Cross-cluster ping

```bash
# Hub -> Spoke
export KUBECONFIG=/home/jkary/labs/hcp/deploy/auth/kubeconfig
oc exec -n cudn-hub-ns hub-test -- ping -c 3 <spoke-cudn-ip>

# Spoke -> Hub
export KUBECONFIG=/home/jkary/labs/hcp/spoke/auth/kubeconfig
oc exec -n cudn-spoke-ns spoke-test -- ping -c 3 <hub-cudn-ip>
```

## Key Learnings

### Network operator requires two separate settings

`additionalRoutingCapabilities.providers: [FRR]` alone only deploys FRR-K8s pods and establishes BGP sessions. It does **not** integrate BGP-learned routes into OVN's data plane.

You must also set `defaultNetwork.ovnKubernetesConfig.routeAdvertisements: Enabled` to deploy the `RouteAdvertisements` CRD. This is the piece that bridges FRR-K8s and OVN — without it, BGP routes appear in the node's kernel routing table but OVN-managed pods cannot use them.

### FRRConfiguration requires `disableMP: true`

When using RouteAdvertisements, the OVN-generated FRRConfiguration and user-created FRRConfiguration must be compatible. The RouteAdvertisements controller requires `disableMP: true` on BGP neighbors — without it, the RouteAdvertisements status shows:

```
configuration error: DisableMP==false not supported
```

### RouteAdvertisements API schema

The API is not what many examples show. The correct structure for 4.20:

```yaml
apiVersion: k8s.ovn.org/v1
kind: RouteAdvertisements
metadata:
  name: cudn-routes
spec:
  advertisements:
  - PodNetwork                              # string array, not objects
  networkSelectors:                          # plural, required
  - networkSelectionType: ClusterUserDefinedNetworks   # required enum
    clusterUserDefinedNetworkSelector:
      networkSelector:
        matchLabels:
          advertise: "true"
  nodeSelector: {}                           # required, use {} for all nodes
  frrConfigurationSelector:
    matchLabels: {}
```

### Pod networking with CUDNs

Pods in a CUDN namespace get two interfaces:
- `eth0` — default cluster network (role: `infrastructure-locked`)
- `ovn-udn1` — CUDN network (role: `primary`, default route)

The pod's default route goes through the CUDN interface, so cross-cluster traffic to other CUDN CIDRs flows through the BGP-routed path automatically.

## Files Reference

| File | Description |
|------|-------------|
| `common/setup.yaml` | Shared setup: c8000v base BGP + cluster FRR enablement |
| `common/manifest/vars.yaml` | Site-specific variables (IPs, ASNs, CIDRs, kubeconfigs) |
| `lab01-cudn-bgp/lab01.yaml` | Lab 01 playbook: create CUDNs and BGP peering |
| `lab01-cudn-bgp/verify.yaml` | Verification: test pods + cross-cluster ping |
| `lab01-cudn-bgp/manifest/c8000v-bgp.cfg` | IOS config for c8000v BGP (AS 64514) |
| `lab01-cudn-bgp/manifest/hub-cudn.yaml` | Hub namespace + ClusterUserDefinedNetwork |
| `lab01-cudn-bgp/manifest/hub-bgp.yaml` | Hub FRRConfiguration + RouteAdvertisements |
| `lab01-cudn-bgp/manifest/spoke-cudn.yaml` | Spoke namespace + ClusterUserDefinedNetwork |
| `lab01-cudn-bgp/manifest/spoke-bgp.yaml` | Spoke FRRConfiguration + RouteAdvertisements |
| `lab01-cudn-bgp/manifest/test-pods.yaml` | Test pods for verification |

## Automated Deployment

```bash
# 1. Edit site-specific variables
vi common/manifest/vars.yaml

# 2. Run common setup (c8000v + cluster FRR enablement)
ansible-playbook common/setup.yaml

# 3. Deploy lab01 (CUDNs + BGP peering)
ansible-playbook lab01-cudn-bgp/lab01.yaml

# 4. Verify cross-cluster connectivity
ansible-playbook lab01-cudn-bgp/verify.yaml

# Individual phases
ansible-playbook lab01-cudn-bgp/lab01.yaml -t hub       # Hub CUDN + BGP only
ansible-playbook lab01-cudn-bgp/lab01.yaml -t spoke     # Spoke CUDN + BGP only
```
