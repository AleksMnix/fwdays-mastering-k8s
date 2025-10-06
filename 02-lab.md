# Lab 02: Deploying nginx and Troubleshooting

## Objective

Deploy a custom nginx deployment with 3 replicas and troubleshoot any issues that arise.

## Task

```bash
sudo kubebuilder/bin/kubectl create deployment nginx --image=nginx --replicas=3
```

## Troubleshooting Journey

### Issue 1: Pods Stuck in Pending State

**Symptom:**
```bash
$ sudo kubebuilder/bin/kubectl get pods -o wide
NAME                    READY   STATUS    RESTARTS   AGE   IP       NODE     NOMINATED NODE   READINESS GATES
nginx-bf5d5cf98-4rvgd   0/1     Pending   0          11s   <none>   <none>   <none>           <none>
nginx-bf5d5cf98-mjp5w   0/1     Pending   0          11s   <none>   <none>   <none>           <none>
nginx-bf5d5cf98-xgfk8   0/1     Pending   0          11s   <none>   <none>   <none>           <none>
```

**Investigation:**
The pods were created successfully but stuck in `Pending` state with no node assignment (`NODE` column shows `<none>`).

**Root Cause Analysis:**

#### 1. No Node Registered

```bash
$ sudo kubebuilder/bin/kubectl get nodes
No resources found
```

The cluster had no worker nodes registered. This was because **kubelet was not running**.

**Why kubelet wasn't running:**
- Kubelet had exited after a previous `sh setup.sh start` command was canceled
- The `start_static` command was run without ensuring all prerequisites were met
- Containerd wasn't running, which caused kubelet to fail with:
  ```
  dial unix /run/containerd/containerd.sock: connect: no such file or directory
  ```

#### 2. Containerd Not Running

Kubelet requires a container runtime (containerd) to manage containers. When kubelet tried to start, it failed because it couldn't connect to containerd's socket.

**Error in logs:**
```
W1006 13:02:44.247371   53567 logging.go:59] [core] [Channel #1 SubChannel #2] grpc: addrConn.createTransport failed to connect to {Addr: "/run/containerd/containerd.sock", ServerName: "%2Frun%2Fcontainerd%2Fcontainerd.sock", }. Err: connection error: desc = "transport: Error while dialing: dial unix /run/containerd/containerd.sock: connect: no such file or directory"
E1006 13:02:44.247992   53567 run.go:74] "command failed" err="failed to run Kubelet: validate service connection: validate CRI v1 runtime API for endpoint \"unix:///run/containerd/containerd.sock\": rpc error: code = Unavailable desc = connection error: desc = \"transport: Error while dialing: dial unix /run/containerd/containerd.sock: connect: no such file or directory\""
```

### Fix 1: Start Containerd

```bash
sudo PATH=$PATH:/opt/cni/bin:/usr/sbin /opt/cni/bin/containerd -c /etc/containerd/config.toml > /tmp/containerd.log 2>&1 &
```

**Verification:**
```bash
$ ps aux | grep "[c]ontainerd -c"
root       54504  0.0  0.0  16384  6272 ?        S<   13:04   0:00 sudo PATH=... /opt/cni/bin/containerd -c /etc/containerd/config.toml
```

### Fix 2: Start Kubelet

```bash
HOST_IP=$(hostname -I | awk '{print $1}')
sudo PATH=$PATH:/opt/cni/bin:/usr/sbin kubebuilder/bin/kubelet \
    --kubeconfig=/var/lib/kubelet/kubeconfig \
    --config=/var/lib/kubelet/config.yaml \
    --root-dir=/var/lib/kubelet \
    --cert-dir=/var/lib/kubelet/pki \
    --tls-cert-file=/var/lib/kubelet/pki/kubelet.crt \
    --tls-private-key-file=/var/lib/kubelet/pki/kubelet.key \
    --hostname-override=$(hostname) \
    --pod-infra-container-image=registry.k8s.io/pause:3.10 \
    --node-ip=$HOST_IP \
    --cloud-provider=external \
    --cgroup-driver=cgroupfs \
    --max-pods=10 \
    --v=1 > /tmp/kubelet.log 2>&1 &
```

**Verification:**
```bash
$ sudo kubebuilder/bin/kubectl get nodes
NAME                STATUS   ROLES    AGE   VERSION
codespaces-87f0bf   Ready    <none>   29s   v1.30.0
```

✅ **Success!** The node is now registered and in `Ready` state.

---

### Issue 2: Pods Still Pending Despite Node Being Ready

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

## Summary: Root Causes and Fixes

| Issue | Root Cause | Fix | Verification |
|-------|-----------|-----|--------------|
| **Pods Pending (No Node)** | Kubelet not running because containerd was not running | Start containerd, then start kubelet | `kubectl get nodes` shows node as Ready |
| **Containerd Socket Missing** | Containerd process was never started | `sudo /opt/cni/bin/containerd -c /etc/containerd/config.toml &` | `ps aux \| grep containerd` shows process |
| **Pods Pending (Node Exists)** | Node had taint `node.cloudprovider.kubernetes.io/uninitialized:NoSchedule` | Remove taint: `kubectl taint nodes <node-name> node.cloudprovider.kubernetes.io/uninitialized:NoSchedule-` | Pods transition to ContainerCreating/Running |

---

## Key Learnings

1. **Container Runtime Dependency**: Kubelet requires a running container runtime (containerd, CRI-O, etc.) before it can start. Always verify the runtime is running.

2. **Node Taints**: When using `--cloud-provider=external`, Kubernetes automatically adds the `node.cloudprovider.kubernetes.io/uninitialized` taint. Without a cloud controller manager, this must be manually removed.

3. **Debugging Workflow**:
   - Check pod status: `kubectl get pods -o wide`
   - Check pod events: `kubectl describe pod <pod-name>`
   - Check node status: `kubectl get nodes`
   - Check node details: `kubectl describe node <node-name>`
   - Check component logs: `tail -f /tmp/kubelet.log`

4. **Startup Order Matters**:
   - containerd → kubelet → node registration → pod scheduling
   - Each step depends on the previous one

5. **Static Pods**: Kubelet can manage static pods from `/etc/kubernetes/manifests/` but still needs containerd to run them.

---

## Cleanup

To clean up the nginx deployment:

```bash
sudo kubebuilder/bin/kubectl delete deployment nginx
```

To clean up the entire cluster:

```bash
sh setup.sh cleanup
```

