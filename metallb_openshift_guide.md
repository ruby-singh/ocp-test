# MetalLB Installation Guide for OpenShift 4.

## Introduction

```
This guide explains how to set up MetalLB on an OpenShift 4.17 bare-metal cluster.
It covers two approaches:
```
1. Operator-based installation (recommended)
2. Manual installation with YAML resources

## 1. Prerequisites

- OpenShift 4.17 cluster (bare-metal or VM lab).
- Cluster admin access (oc CLI logged in).
- A pool of spare IP addresses in the same Layer2 network as worker nodes (not managed by DHCP).
Example: 192.168.1.240–192.168.1.250.
- Access to the cluster to create custom resources.

## 2. Operator-based Installation - Step 1: Create Namespace

```
oc create namespace metallb-system
```
## Step 2: Install MetalLB Operator

```
Option A – Web Console:
```
1. OpenShift Console → Operators → OperatorHub.
2. Search for MetalLB.
3. Install into namespace metallb-system.

```
Option B – CLI YAML:
```
```
apiVersion: operators.coreos.com/v
kind: OperatorGroup
metadata:
name: metallb-operatorgroup
namespace: metallb-system
spec:
targetNamespaces:
```
- metallb-system
---
apiVersion: operators.coreos.com/v1alpha
kind: Subscription
metadata:
name: metallb-subscription
namespace: metallb-system
spec:
channel: stable
name: metallb-operator
source: redhat-operators
sourceNamespace: openshift-marketplace

## Step 3: Deploy a MetalLB Instance

```
apiVersion: metallb.io/v1beta
kind: MetalLB
metadata:
name: metallb
namespace: metallb-system
```
## Step 4: Configure IPAddressPool


```
apiVersion: metallb.io/v1beta
kind: IPAddressPool
metadata:
name: my-ip-pool
namespace: metallb-system
spec:
addresses:
```
- 192.168.1.240-192.168.1.

## Step 5: Configure L2Advertisement

```
apiVersion: metallb.io/v1beta
kind: L2Advertisement
metadata:
name: l2adv
namespace: metallb-system
spec:
ipAddressPools:
```
- my-ip-pool

## Step 6: Test with LoadBalancer Service

```
oc new-project test-lb
oc create deployment nginx --image=nginx --replicas=
oc expose deployment nginx --port=
oc expose deployment nginx --type=LoadBalancer --port=
oc get svc
curl http://192.168.1.
```
## 3. Manual Installation Steps (Alternative)

#### If you don’t use the Operator, you can deploy MetalLB by applying manifests manually.

### Step 1: Create Namespace

```
oc create namespace metallb-system
```
### Step 2: Deploy MetalLB Components

```
oc apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.12/config/manifests/metallb-native.yaml -n metallb-system
```
### Step 3: Configure IPAddressPool

```
apiVersion: metallb.io/v1beta
kind: IPAddressPool
metadata:
name: my-ip-pool
namespace: metallb-system
spec:
addresses:
```
- 192.168.1.240-192.168.1.

### Step 4: Configure L2Advertisement

```
apiVersion: metallb.io/v1beta
kind: L2Advertisement
metadata:
name: l2adv
namespace: metallb-system
spec:
ipAddressPools:
```
- my-ip-pool


### Step 5: Test with LoadBalancer Service

```
oc new-project test-lb
oc create deployment nginx --image=nginx --replicas=
oc expose deployment nginx --port=
oc expose deployment nginx --type=LoadBalancer --port=
oc get svc
curl http://192.168.1.
```
## 4. Monitoring & Troubleshooting

- Check pods: oc get pods -n metallb-system
- Service details: oc describe svc <service-name>
- Logs: oc logs -n metallb-system deploy/controller

## 5. Optional: BGP Mode

#### For routed networks, configure: - BGPPeer - BGPAdvertisement Instead of L2 mode.


