# Lab 02: Deploying nginx and Troubleshooting

## Objective

Deploy a custom nginx deployment with 3 replicas and troubleshoot any issues that arise.

## Task

```bash
sudo kubebuilder/bin/kubectl create deployment nginx --image=nginx --replicas=3
```

---

## Troubleshooting: Pods Still Pending Despite Node Being Ready

**Symptom:**
Even after the node was registered and showing as `Ready`, the nginx pods remained in `Pending` state:

```bash
$ sudo kubebuilder/bin/kubectl get pods -o wide
NAME                    READY   STATUS    RESTARTS   AGE    IP       NODE     NOMINATED NODE   READINESS GATES
nginx-bf5d5cf98-4rvgd   0/1     Pending   0          104s   <none>   <none>   <none>           <none>
nginx-bf5d5cf98-mjp5w   0/1     Pending   0          104s   <none>   <none>   <none>           <none>
nginx-bf5d5cf98-xgfk8   0/1     Pending   0          104s   <none>   <none>   <none>           <none>
```

**Investigation:**

Check pod events:
```bash
$ sudo kubebuilder/bin/kubectl describe pod nginx-bf5d5cf98-4rvgd | grep -A 10 "Events:"
Events:
  Type     Reason            Age                  From               Message
  ----     ------            ----                 ----               -------
  Warning  FailedScheduling  111s                 default-scheduler  no nodes available to schedule pods
  Warning  FailedScheduling  108s (x2 over 110s)  default-scheduler  no nodes available to schedule pods
```

**Root Cause:**
The scheduler reported **"no nodes available to schedule pods"** even though a node existed and was `Ready`. This indicated the node might have **taints** preventing pod scheduling.

**Verification:**
```bash
$ sudo kubebuilder/bin/kubectl describe node codespaces-87f0bf | grep -A 5 "Taints:"
Taints:             node.cloudprovider.kubernetes.io/uninitialized=true:NoSchedule
Unschedulable:      false
Lease:
  HolderIdentity:  codespaces-87f0bf
  AcquireTime:     <unset>
  RenewTime:       Mon, 06 Oct 2025 13:08:24 +0000
```

**Root Cause Confirmed:**
The node had a taint `node.cloudprovider.kubernetes.io/uninitialized=true:NoSchedule`. This taint is automatically added by Kubernetes when `--cloud-provider=external` is specified, and it prevents pods from being scheduled until the cloud controller manager removes it.

Since we're running a minimal setup without a cloud controller manager, we need to manually remove this taint.

### Fix 3: Remove the Node Taint

```bash
sudo kubebuilder/bin/kubectl taint nodes codespaces-87f0bf node.cloudprovider.kubernetes.io/uninitialized:NoSchedule-
```

Output:
```
node/codespaces-87f0bf untainted
```

**Immediate Effect:**
```bash
$ sudo kubebuilder/bin/kubectl get pods -o wide
NAME                    READY   STATUS              RESTARTS   AGE     IP       NODE                NOMINATED NODE   READINESS GATES
nginx-bf5d5cf98-4rvgd   0/1     ContainerCreating   0          2m34s   <none>   codespaces-87f0bf   <none>           <none>
nginx-bf5d5cf98-mjp5w   0/1     ContainerCreating   0          2m34s   <none>   codespaces-87f0bf   <none>           <none>
nginx-bf5d5cf98-xgfk8   0/1     ContainerCreating   0          2m34s   <none>   codespaces-87f0bf   <none>           <none>
```

✅ Pods transitioned from `Pending` to `ContainerCreating` immediately after removing the taint!

---

## Final Verification

After waiting for the containers to be created:

```bash
$ sudo kubebuilder/bin/kubectl get pods -o wide
NAME                    READY   STATUS    RESTARTS   AGE     IP          NODE                NOMINATED NODE   READINESS GATES
nginx-bf5d5cf98-4rvgd   1/1     Running   0          2m52s   10.22.0.7   codespaces-87f0bf   <none>           <none>
nginx-bf5d5cf98-mjp5w   1/1     Running   0          2m52s   10.22.0.8   codespaces-87f0bf   <none>           <none>
nginx-bf5d5cf98-xgfk8   1/1     Running   0          2m52s   10.22.0.6   codespaces-87f0bf   <none>           <none>
```

```bash
$ sudo kubebuilder/bin/kubectl get deployments
NAME    READY   UP-TO-DATE   AVAILABLE   AGE
nginx   3/3     3            3           3m
```

🎉 **Success!** All 3 nginx replicas are running with IPs assigned.

---

---

## Summary

### Root Cause
Node had taint `node.cloudprovider.kubernetes.io/uninitialized:NoSchedule` that prevented pod scheduling.

### Fix
Remove the taint manually:
```bash
kubectl taint nodes <node-name> node.cloudprovider.kubernetes.io/uninitialized:NoSchedule-
```

### Verification
Pods transition to `ContainerCreating` → `Running`

---

## Key Learnings

### 1. Node Taints and Pod Scheduling
When using `--cloud-provider=external`, Kubernetes automatically adds the `node.cloudprovider.kubernetes.io/uninitialized:NoSchedule` taint to prevent pod scheduling until a cloud controller initializes the node. Without a real cloud controller manager, this taint must be **manually removed**.

### 2. Debugging Workflow for Pending Pods

**Step 1:** Check pod status
```bash
kubectl get pods -o wide
```
Look for: `STATUS`, `NODE` assignment, `IP` address

**Step 2:** Describe the pod to see events
```bash
kubectl describe pod <pod-name>
```
Look for: Scheduling events, warnings, error messages

**Step 3:** Check node availability and status
```bash
kubectl get nodes
```
Verify: Node is present and in `Ready` state

**Step 4:** Inspect node for taints and conditions
```bash
kubectl describe node <node-name> | grep -A 5 "Taints:"
```
Look for: Any `NoSchedule` or `NoExecute` taints

### 3. Common Reasons for Pending Pods

| Reason | How to Identify | Solution |
|--------|----------------|----------|
| **No nodes available** | `kubectl get nodes` returns no nodes | Start/register kubelet |
| **Node taints** | `kubectl describe node` shows taints | Remove inappropriate taints |
| **Resource constraints** | Pod events show "Insufficient cpu/memory" | Add resources or scale nodes |
| **Node selector mismatch** | Pod has nodeSelector that no node matches | Fix labels or nodeSelector |
| **PVC issues** | Pod events show volume/PVC errors | Fix PersistentVolumeClaim |

### 4. Understanding Taints and Tolerations

**Taints** prevent pods from being scheduled on nodes unless the pods have matching **tolerations**.

**Common taints:**
- `node.cloudprovider.kubernetes.io/uninitialized:NoSchedule` - Cloud provider not initialized
- `node.kubernetes.io/not-ready:NoSchedule` - Node not ready
- `node.kubernetes.io/unreachable:NoSchedule` - Node unreachable
- `node.kubernetes.io/disk-pressure:NoSchedule` - Disk pressure on node

**View node taints:**
```bash
kubectl describe node <node-name> | grep Taints
```

**Remove a taint:**
```bash
kubectl taint nodes <node-name> <taint-key>:<effect>-
# Example: kubectl taint nodes worker-1 node.cloudprovider.kubernetes.io/uninitialized:NoSchedule-
```

---

## Quick Reference

### Troubleshooting Commands

```bash
# Check deployment status
kubectl get deployments

# Check pod status with details
kubectl get pods -o wide

# Get detailed pod information
kubectl describe pod <pod-name>

# Check pod events only
kubectl get events --field-selector involvedObject.name=<pod-name>

# Check node status
kubectl get nodes

# Check node details and taints
kubectl describe node <node-name>

# Check scheduler logs (if running)
kubectl logs -n kube-system <scheduler-pod-name>
```

### Cleanup

Clean up the nginx deployment:
```bash
sudo kubebuilder/bin/kubectl delete deployment nginx
```

Clean up the entire cluster:
```bash
sh setup.sh cleanup
```

