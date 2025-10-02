# Kubernetes Troubleshooting: Pod Deployment Stuck in Pending

## Scenario

After successfully starting the Kubernetes cluster with `./setup.sh start`, you attempt to deploy a simple nginx application:

```bash
sudo kubebuilder/bin/kubectl create deploy demo --image nginx
```

## Problem: Pod Stuck in Pending Status

The deployment is created, but the pod remains in `Pending` status indefinitely:

```bash
$ sudo kubebuilder/bin/kubectl get all -A
NAMESPACE   NAME                        READY   STATUS    RESTARTS   AGE
default     pod/demo-677cfb9d49-cmzr5   0/1     Pending   0          112s

NAMESPACE   NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
default     service/kubernetes   ClusterIP   10.0.0.1     <none>        443/TCP   5m52s

NAMESPACE   NAME                   READY   UP-TO-DATE   AVAILABLE   AGE
default     deployment.apps/demo   0/1     1            0           112s

NAMESPACE   NAME                              DESIRED   CURRENT   READY   AGE
default     replicaset.apps/demo-677cfb9d49   1         1         0       112s
```

**Key Indicators**:

- Pod status: `Pending` (not `Running` or `ContainerCreating`)
- READY: `0/1` (container not started)
- Deployment shows `0` AVAILABLE replicas

## Troubleshooting Steps

### Step 1: Describe the Pod to Find the Issue

```bash
sudo kubebuilder/bin/kubectl describe pod demo-677cfb9d49-cmzr5 | grep -A 10 "Events:"
```

**Output**:

```bash
Events:
  Type     Reason            Age    From               Message
  ----     ------            ----   ----               -------
  Warning  FailedScheduling  3m58s  default-scheduler  0/1 nodes are available: 1 node(s) had untolerated taint {node.cloudprovider.kubernetes.io/uninitialized: true}. preemption: 0/1 nodes are available: 1 Preemption is not helpful for scheduling.
```

**Analysis**: The error message clearly indicates:

- **Problem**: Node has an untolerated taint
- **Taint**: `node.cloudprovider.kubernetes.io/uninitialized=true:NoSchedule`
- **Impact**: Scheduler cannot place the pod on any available node

### Step 2: Verify Node Taints

```bash
sudo kubebuilder/bin/kubectl describe node $(hostname) | grep -A 5 "Taints:"
```

**Output**:

```bash
Taints:             node.cloudprovider.kubernetes.io/uninitialized=true:NoSchedule
Unschedulable:      false
Lease:
  HolderIdentity:  codespaces-87f0bf
  AcquireTime:     <unset>
  RenewTime:       Thu, 02 Oct 2025 10:26:08 +0000
```

**Confirmed**: The node has the cloud provider taint that blocks scheduling.

### Step 3: Check Node Status

```bash
sudo kubebuilder/bin/kubectl get nodes
```

**Output**:

```bash
NAME                STATUS   ROLES    AGE     VERSION
codespaces-87f0bf   Ready    <none>   3m25s   v1.30.0
```

**Important Note**: The node shows `STATUS: Ready`, which is misleading. The node is ready from a kubelet perspective, but the taint prevents any workload from being scheduled.

### Step 4: Apply the Fix

Remove the cloud provider taint:

```bash
sudo kubebuilder/bin/kubectl taint nodes --all node.cloudprovider.kubernetes.io/uninitialized:NoSchedule-
```

**Output**:

```bash
node/codespaces-87f0bf untainted
```

### Step 5: Verify Pod Scheduling

Immediately after removing the taint, the scheduler picks up the pod:

```bash
sudo kubebuilder/bin/kubectl get pods -w
```

**Output sequence**:

```bash
NAME                    READY   STATUS              RESTARTS   AGE
demo-677cfb9d49-cmzr5   0/1     Pending             0          4m30s
demo-677cfb9d49-cmzr5   0/1     ContainerCreating   0          4m30s
demo-677cfb9d49-cmzr5   1/1     Running             0          4m50s
```

**Progress**:

1. `Pending` → `ContainerCreating`: Pod is scheduled to the node
2. `ContainerCreating` → `Running`: Container image pulled and started

### Step 6: Verify Full Deployment

```bash
sudo kubebuilder/bin/kubectl get all -A
```

**Output**:

```bash
NAMESPACE   NAME                        READY   STATUS    RESTARTS   AGE     IP          NODE
default     pod/demo-677cfb9d49-cmzr5   1/1     Running   0          4m50s   10.22.0.2   codespaces-87f0bf

NAMESPACE   NAME                   READY   UP-TO-DATE   AVAILABLE   AGE
default     deployment.apps/demo   1/1     1            1           4m50s

NAMESPACE   NAME                              DESIRED   CURRENT   READY   AGE
default     replicaset.apps/demo-677cfb9d49   1         1         1       4m50s
```

**Success Indicators**:

- ✅ Pod status: `Running`
- ✅ Pod READY: `1/1`
- ✅ Pod has IP address: `10.22.0.2`
- ✅ Deployment AVAILABLE: `1`
- ✅ ReplicaSet READY: `1`

## Understanding the Timeline

```bash
T+0s:    kubebuilder/bin/kubectl create deploy demo --image nginx
         ↓ Deployment created
         ↓ ReplicaSet created
         ↓ Pod created (status: Pending)

T+0s-4m: Pod stuck in Pending
         ↓ Scheduler evaluates node
         ↓ Node has untolerated taint
         ↓ Scheduler cannot place pod
         ↓ Event: FailedScheduling

T+4m30s: kubebuilder/bin/kubectl taint nodes --all node.cloudprovider.kubernetes.io/uninitialized:NoSchedule-
         ↓ Taint removed
         ↓ Scheduler immediately notices node is now available
         ↓ Pod status: ContainerCreating

T+4m35s: Kubelet pulls pause container image
         ↓ Kubelet pulls nginx:latest image (72MB)
         ↓ Kubelet creates container
         ↓ Container starts

T+4m50s: Pod status: Running (healthy)
```

## Root Cause

**Cloud Provider Taint**: When using `--cloud-provider=external`, Kubernetes automatically taints the node with `node.cloudprovider.kubernetes.io/uninitialized=true:NoSchedule`. This taint is meant to be removed by an external cloud controller manager once it initializes the node. However, in our local setup without a real cloud provider, this taint is never removed automatically.

## Permanent Fix

The fix has been added to `setup.sh` to automatically remove the taint after node registration:

```bash
# Wait for node to be registered
echo "Waiting for node registration..."
sleep 5

# Label the node
NODE_NAME=$(hostname)
sudo kubebuilder/bin/kubectl label node "$NODE_NAME" node-role.kubernetes.io/master="" --overwrite || true

# Remove cloud provider taint to allow pod scheduling
echo "Removing cloud provider taint..."
sudo kubebuilder/bin/kubectl taint nodes --all node.cloudprovider.kubernetes.io/uninitialized:NoSchedule- 2>/dev/null || true
```

## Key Learnings

1. **Node Ready ≠ Workload Ready**: A node can show as `Ready` but still be unable to accept pods due to taints.

2. **Always Check Events**: The `kubectl describe pod` events are the first place to look when pods are stuck.

3. **Taints Block Scheduling**: Taints with `NoSchedule` effect prevent new pods from being scheduled (but don't evict existing ones).

4. **Cloud Provider Taints**: When using `--cloud-provider=external`, Kubernetes expects a cloud controller manager to remove the initialization taint.

5. **Immediate Scheduling**: Once taints are removed, the scheduler immediately picks up pending pods.

## Verification Commands

After the fix is applied, verify the cluster is working:

```bash
# Check node has no taints
sudo kubebuilder/bin/kubectl describe node $(hostname) | grep Taints
# Expected: Taints: <none>

# Create a test pod
sudo kubebuilder/bin/kubectl run test-nginx --image=nginx:alpine

# Verify pod is scheduled and running
sudo kubebuilder/bin/kubectl get pods -o wide

# Clean up
sudo kubebuilder/bin/kubectl delete pod test-nginx
sh setup.sh cleanup
```

---

## Cleanup Troubleshooting: Busy Mount Points

### Problem: Cannot Remove Kubelet Directories

After running `./setup.sh cleanup`, you may encounter errors when trying to remove kubelet data:

```bash
$ ./setup.sh cleanup
...
Cleaning up...
rm: cannot remove '/var/lib/kubelet/pods/2ed2d72a-af64-4cdb-bc6a-294afd28d580/volumes/kubernetes.io~projected/kube-api-access-pm55s': Device or resource busy
```

### Root Cause

**Projected Volumes Still Mounted**: Kubernetes uses projected volumes to inject service account tokens, ConfigMaps, and secrets into pods. These volumes are mounted by the kernel and remain mounted even after the kubelet and containerd are stopped. The Linux kernel keeps these mounts active, preventing their removal.

### Troubleshooting Steps

#### Step 1: Identify Busy Mounts

Check which volumes are still mounted:

```bash
mount | grep kubelet
```

**Typical Output**:

```bash
tmpfs on /var/lib/kubelet/pods/2ed2d72a-af64-4cdb-bc6a-294afd28d580/volumes/kubernetes.io~projected/kube-api-access-pm55s type tmpfs (rw,relatime)
```

#### Step 2: Understand the Issue

- **Mount Type**: `tmpfs` (temporary filesystem in memory)
- **Mount Point**: Pod-specific projected volume
- **Status**: Active mount prevents directory deletion
- **Impact**: `rm -rf` fails with "Device or resource busy"

### The Fix

The improved cleanup function in `setup.sh` now handles busy mounts:

```bash
cleanup() {
    stop
    echo "Cleaning up..."

    # Force unmount any busy projected volumes
    echo "Unmounting projected volumes..."
    sudo umount -l /var/lib/kubelet/pods/*/volumes/kubernetes.io~projected/* 2>/dev/null || true
    sleep 1

    # Remove kubelet data (with fallback for busy mounts)
    echo "Removing kubelet data..."
    if ! sudo rm -rf /var/lib/kubelet/* 2>/dev/null; then
        # If removal fails due to busy mounts, clean everything except busy directories
        echo "Warning: Some volumes are busy, cleaning other directories..."
        sudo find /var/lib/kubelet -mindepth 1 -maxdepth 1 ! -name "pods" -delete 2>/dev/null || true
        # Try to clean pod directories that aren't busy
        for pod_dir in /var/lib/kubelet/pods/*; do
            if [ -d "$pod_dir" ]; then
                sudo rm -rf "$pod_dir" 2>/dev/null || echo "Skipping busy mount in $(basename $pod_dir)"
            fi
        done
    fi

    # Clean up other directories
    sudo rm -rf ./etcd
    sudo rm -rf /run/containerd/*
    sudo rm -f /tmp/sa.key /tmp/sa.pub /tmp/token.csv /tmp/ca.key /tmp/ca.crt

    echo "Cleanup complete"
}
```

### How the Fix Works

1. **Lazy Unmount**: `umount -l` performs a lazy unmount, detaching the filesystem immediately but cleaning up references when they're no longer busy

   ```bash
   sudo umount -l /var/lib/kubelet/pods/*/volumes/kubernetes.io~projected/* 2>/dev/null || true
   ```

2. **Grace Period**: `sleep 1` gives the kernel time to release mount references

3. **Fallback Strategy**: If `rm -rf` still fails, the script:
   - Cleans non-pod directories first
   - Iterates through each pod directory individually
   - Skips directories that are still busy
   - Continues with the rest of the cleanup

### Verification

After cleanup, verify all resources are removed:

```bash
# Check for remaining mounts
mount | grep kubelet
# Expected: (no output)

# Check for remaining files
ls -la /var/lib/kubelet/
# Expected: directory empty or doesn't exist

# Verify cluster is stopped
ps aux | grep -E "kube-apiserver|kubelet|etcd" | grep -v grep
# Expected: (no output)
```

### Key Learnings

1. **Mount Points vs Files**: Mounted filesystems must be unmounted before their directories can be removed.

2. **Lazy Unmount**: The `-l` flag for `umount` is useful when processes might still have references to the mount.

3. **Graceful Degradation**: Cleanup scripts should handle partial failures and clean up as much as possible.

4. **Projected Volumes**: Kubernetes projected volumes combine multiple sources (secrets, ConfigMaps, tokens) into a single volume mount.

5. **Background Processes**: Even after stopping services, the kernel may maintain references to resources.
