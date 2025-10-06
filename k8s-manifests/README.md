# Kubernetes Control Plane Manifests

This directory contains YAML manifests for Kubernetes control plane components. These manifests are used as templates and will be processed by the `setup.sh` script to generate static pod definitions.

## Components

### 1. etcd.yaml
- **Purpose**: Distributed key-value store used by Kubernetes to store all cluster data
- **Port**: 2379 (client), 2380 (peer)
- **Data Directory**: `/var/lib/etcd`
- **Health Check**: HTTP health endpoint

### 2. kube-apiserver.yaml
- **Purpose**: API server that exposes the Kubernetes API
- **Port**: 6443 (HTTPS)
- **Dependencies**: etcd
- **Features**:
  - Token-based authentication
  - External cloud provider support
  - Service account token signing

### 3. kube-scheduler.yaml
- **Purpose**: Watches for newly created pods and assigns them to nodes
- **Port**: 10259 (HTTPS)
- **Dependencies**: kube-apiserver
- **Features**:
  - Leader election disabled (single-node setup)
  - Health and readiness probes

### 4. kube-controller-manager.yaml
- **Purpose**: Runs controller processes that regulate the state of the cluster
- **Port**: 10257 (HTTPS)
- **Dependencies**: kube-apiserver
- **Features**:
  - External cloud provider support
  - Service account token controller
  - Various built-in controllers (replication, namespace, serviceaccount, etc.)

## Usage

These manifests are templates that contain `HOST_IP` placeholders. The `setup.sh` script:

1. Detects the host IP address
2. Replaces `HOST_IP` with the actual IP in each manifest
3. Copies the processed manifests to `/etc/kubernetes/manifests/`
4. Kubelet automatically creates and manages these static pods

## Configuration

All manifests use:
- **hostNetwork**: `true` - Pods use the host's network namespace
- **priorityClassName**: `system-cluster-critical` - Ensures high priority scheduling
- **Resource Requests**: Set reasonable CPU/memory requests for control plane components

## Notes

- These are static pod manifests managed by kubelet
- Changes to files in `/etc/kubernetes/manifests/` are automatically detected by kubelet
- To update a component, modify the template here and re-run `setup.sh start`
- All certificates and keys are stored in `/etc/kubernetes/pki/`

