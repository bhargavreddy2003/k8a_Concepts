# Reverse-Engineering Kubernetes Failures: Bottom‑Up Debugging (Node → CRI → Kubelet → Control Plane)

> Goal: Learn how to debug Kubernetes **from the bottom up** as if you are reverse‑engineering failures: start at the host and container runtime (CRI), move through **kubelet**, then up to the **Kubernetes API and controllers**.
>
> This guide assumes kubeadm‑style clusters, containerd as runtime, and `crictl` for node‑level inspection.[cite:82][cite:75]

---

## 0. Mental Model for Bottom‑Up Debugging

When something breaks, you can think in **three vertical layers**:

1. **Host / OS / Network / Storage**
2. **Container runtime (CRI) and pods/containers on that node**
3. **Kubelet → Kubernetes control plane (API, scheduler, controllers)**[cite:59][cite:63]

Reverse‑engineering flow:

1. Pick a failing **Pod / Service / Node**.
2. Drop down to the **node** that owns it.
3. Use **CRI tools** (`crictl`, `ctr`) and Linux commands to see what is actually running.
4. Work **upwards**: kubelet logs, then API objects, then controllers.

This file is structured that way.

---

## 1. Node & Host-Level Debugging (Bottom Layer)

### 1.1 Node Health Snapshot

Start with a quick host snapshot when anything on that node looks off:

```bash
# CPU, load, memory, IO
uptime
free -m
iostat -x 1 5     # if installed
vmstat 1 10

# Disk usage
df -h
lsblk

# Processes
ps aux --sort=-%mem | head -n 20
ps aux --sort=-%cpu | head -n 20

# Network
ip addr
ip route
ss -tulpn | head -n 30
```

Key things to look for:

- Disk full (especially `/var/lib/kubelet`, `/var/lib/containerd`, `/var/lib/etcd`).
- Memory pressure (swap usage, high `kswapd` activity).
- Network interfaces missing or down.
- IP routing that does **not** cover the Pod CIDR.

### 1.2 Systemd Services: kubelet & runtime

```bash
sudo systemctl status kubelet
sudo systemctl status containerd

sudo journalctl -u kubelet -f
sudo journalctl -u containerd -f
```

Common patterns:

- Kubelet logs show repeated failures to connect to API server, or CNI errors.
- Containerd logs show image pull failures or CRI API issues.[cite:63]

If kubelet **won’t start**:

- Check config file: `/var/lib/kubelet/config.yaml` (or distro path).
- Check bootstrap kubeconfig: `/etc/kubernetes/kubelet.conf`.
- Check cgroup driver mismatch: ensure containerd uses `SystemdCgroup = true` and kubelet `cgroupDriver: systemd`.

---

## 2. CRI / Runtime-Level Debugging with `crictl` & `ctr`

> Use `crictl` when `kubectl` is not working or when you suspect a **control-plane issue** but want to see if containers actually run on the node.[cite:75][cite:82]

### 2.1 Configure `crictl`

Create `/etc/crictl.yaml` (for containerd):

```yaml
runtime-endpoint: unix:///run/containerd/containerd.sock
image-endpoint: unix:///run/containerd/containerd.sock
timeout: 10
debug: true
```

Verify:

```bash
sudo crictl info
```

### 2.2 Inspect Pods & Containers (Node View)

```bash
# List pod sandboxes known to CRI
sudo crictl pods

# Filter for specific namespace or pod name
sudo crictl pods --name <pod-name>

# List containers (including stopped)
sudo crictl ps -a

# Inspect specific container
sudo crictl inspect <container-id>

# Container logs (if kubectl logs is broken)
sudo crictl logs <container-id>
```

Use cases:

- Pod state mismatch: Kubernetes says `Running` but container is repeatedly OOMKilled → `crictl ps -a` and `crictl logs` show actual crash.
- API is down: `kubectl` can’t query Pod, but `crictl` can still show containers and logs.[cite:82]

### 2.3 Deep Container List with `ctr`

Some runtimes (containerd) hide infra containers in `crictl ps`; if needed:

```bash
sudo ctr -n k8s.io c ls
```

This shows **all containers**, including the pause/infra containers that back Pod sandboxes.[cite:77]

### 2.4 Reverse-Engineering Pod Issues from CRI Side

Example workflow:

1. You know Pod `payment-api-xyz` is problematic.
2. From control plane: find its node:

   ```bash
   kubectl get pod payment-api-xyz -o wide -n payments
   ```

3. SSH to that node; map Pod → CRI sandbox:

   ```bash
   sudo crictl pods --name payment-api-xyz
   sudo crictl ps -a | grep payment-api-xyz
   ```

4. Inspect and fetch logs:

   ```bash
   sudo crictl inspect <container-id> | less
   sudo crictl logs <container-id>
   ```

5. Compare container state vs K8s Pod status; if they differ, suspect **kubelet sync lag** or **API connectivity** problems.

---

## 3. Kubelet Debugging – The Bridge Layer

> Kubelet is the **bridge** between the control plane and CRI. Reverse‑engineering failures here explains most Pod scheduling/runtime anomalies.

### 3.1 Kubelet Configuration & Identity

Check config and identity:

```bash
sudo cat /var/lib/kubelet/config.yaml
sudo cat /etc/kubernetes/kubelet.conf  # kubeconfig to API

# Client cert info
openssl x509 -in /var/lib/kubelet/pki/kubelet-client-current.pem -noout -text | grep -E 'Subject:|Not Before|Not After'
```

Look for:

- Wrong `clusterDomain`, Pod CIDR, or DNS IP.
- Wrong API server URL or CA bundle.
- Expired kubelet client cert → node repeatedly `NotReady`.

### 3.2 Kubelet Logs – Common Patterns

```bash
sudo journalctl -u kubelet -f
```

Watch for:

- **API errors**: `x509: certificate signed by unknown authority`, `Unauthorized`, `EOF`.
- **CNI errors**: `Failed to create pod sandbox`, `Error adding network`.
- **Volume mount errors**: CSI timeouts, `MountVolume.SetUp failed`.
- **Probe failures**: liveness/readiness causing restarts.

### 3.3 Reverse-Engineering From Kubelet Upwards

Suppose a Pod is stuck `ContainerCreating`:

1. `kubectl describe pod` → events show `FailedCreatePodSandBox`.
2. On node: `sudo journalctl -u kubelet | grep "FailedCreatePodSandBox"`.
3. You see detailed CNI failure messages (e.g., IPAM error, missing binary).
4. Use CNI plugin logs & host networking tools to finish debugging (see next section).[cite:69]

If Pod is `Running` but not reachable via Service:

- Kubelet logs might show hairpin mode issues or host networking problems; official docs suggest checking `hairpin-mode` logs in kubelet.[cite:69]

---

## 4. Networking Debugging Bottom-Up

> Reverse‑engineering network failures means starting at **pod NICs and host routes** and working up to Services and Ingress.

### 4.1 Pod Network Namespace & Interfaces

From a debug pod scheduled on the same node:

```bash
# Inside debug pod
ip addr
ip route

# Test connectivity to another pod IP
ping <other-pod-ip>
curl http://<other-pod-ip>:<port>
```

From the node:

```bash
# Show node routes and verify Pod CIDR present
ip route

# Show CNI-created interfaces
ip link
```

Edge cases:

- Missing Pod CIDR route → CNI misconfiguration.
- Wrong MTU causing weird intermittent failures.

### 4.2 Service & kube-proxy Path

On a node:

```bash
# iptables mode
sudo iptables -t nat -L KUBE-SERVICES -n --line-numbers
sudo iptables -t nat -L KUBE-SVC-XXXXXX -n

# IPVS mode
sudo ipvsadm -Ln
```

Verify that:

- The **ClusterIP** appears in KUBE-SERVICES and points to correct Endpoint chains.
- Endpoint chains point to Pod IPs that are actually **Running/Ready** (cross-check with `crictl` or `kubectl`).

If you suspect kube-proxy itself:

```bash
kubectl get daemonset kube-proxy -n kube-system -o wide
kubectl logs daemonset/kube-proxy -n kube-system
```

### 4.3 Using `kubectl debug node` for Deep Node Inspection

If you can’t SSH to nodes, you can still debug them via `kubectl debug node`:[cite:76]

```bash
kubectl debug node/<node-name> -it --image=ubuntu
# You now have a pod running with access to the node's filesystem at /host

root@debugger:/# chroot /host
root@node:/# ip addr
root@node:/# iptables -t nat -L -n
```

This lets you run **all usual Linux networking tools** (`ip`, `ss`, `tcpdump`, `mtr`, etc.) from within the cluster.

---

## 5. Storage & Volume Debugging Bottom-Up

### 5.1 Node Disk & Filesystem

On node:

```bash
df -h
lsblk
sudo ls -R /var/lib/kubelet/pods | head
```

Look for:

- Disk full → kubelet cannot create volumes or write logs.
- Missing underlying devices for CSI volumes.

### 5.2 CSI & Volume Mount Failures

From Kubelet logs:

```bash
sudo journalctl -u kubelet | grep -i 'MountVolume' | tail
```

From Kubernetes view:

```bash
kubectl describe pod <pod>  # look at volume-related events
kubectl get pvc -n <ns>
kubectl describe pvc <pvc-name> -n <ns>
```

Reverse‑engineering path:

1. PVC `Pending` → PV/StorageClass or dynamic provisioning issue (controller manager / CSI controller).
2. PVC `Bound` but pod `ContainerCreating` → node‑side CSI / mount problem (kubelet/CSI node plugin).
3. Node plugin logs in CSI DaemonSet.

---

## 6. Control Plane Debugging From Node Perspective

> When `kubectl` is flaky or API is slow/unavailable, confirm control-plane health **from node and pod shells**.

### 6.1 Basic Control-Plane Health

From any machine with `kubectl`:

```bash
kubectl get nodes
kubectl get pods -n kube-system
kubectl cluster-info
kubectl cluster-info dump   # full diagnostics dump[cite:59]
```

From a node without `kubectl`:

```bash
# curl API server directly (using admin.conf)
export KUBECONFIG=/etc/kubernetes/admin.conf
kubectl cluster-info
# or
curl -k https://<apiserver>:6443/healthz
```

If API is down, but static Pods are running:

```bash
sudo ls /etc/kubernetes/manifests
sudo crictl ps | grep kube-apiserver
sudo crictl logs <apiserver-container-id>
```

Look for:

- etcd connection failure (`connection refused`, `deadline exceeded`).
- Cert or key mismatch (`x509` errors).

### 6.2 etcd from Node Perspective

Within etcd Pod or node:

```bash
# In etcd pod
kubectl exec -n kube-system <etcd-pod> -- \
  etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint health
```

If unreachable:

- Check etcd Pod logs via `kubectl logs` or `crictl logs`.
- Check disk space and IOPS.

---

## 7. Full Reverse-Debug Flow: Example Scenarios

### 7.1 Scenario: "My service is down" (Unknown Cause)

1. **User symptom**: 5xx from external LB.
2. **Top-down view** (quick):

   ```bash
   kubectl get pods,svc,ingress -n prod-api
   kubectl get events -n prod-api --sort-by=.metadata.creationTimestamp
   ```

3. **Choose a failing pod**, e.g., `orders-api-123`.
4. **Node location**:

   ```bash
   kubectl get pod orders-api-123 -o wide -n prod-api
   # Note NODE column and POD IP
   ```

5. **Host & kubelet** on that node:

   ```bash
   ssh <node>
   sudo systemctl status kubelet containerd
   sudo journalctl -u kubelet -f
   ```

6. **CRI check**:

   ```bash
   sudo crictl pods --name orders-api-123
   sudo crictl ps -a | grep orders-api-123
   sudo crictl logs <container-id>
   ```

7. **Networking**:

   - Inside debug pod on same node:

     ```bash
     kubectl debug node/<node> -it --image=ubuntu
     root@node:/# ip addr; ip route
     root@node:/# curl http://<orders-pod-ip>:8080/health
     ```

8. **Service path**:

   ```bash
   kubectl describe svc orders-api -n prod-api
   kubectl get endpointslices -l kubernetes.io/service-name=orders-api -n prod-api
   ```

9. Based on where it fails, you know **which layer** broke:

   - Pod not healthy → app or runtime.
   - Pod healthy, but Service/EpSlice wrong → label/selectors.
   - Service correct, but external 5xx → ingress / LB / mesh.

### 7.2 Scenario: "kubectl is dead, but apps still receive traffic"

1. From node:

   ```bash
   sudo crictl ps              # are app containers running?
   sudo crictl logs <id>       # check app logs
   ```

2. Check API from internal node:

   ```bash
   curl -k https://<apiserver>:6443/healthz
   ```

3. If API unhealthy but apps fine, you’ve isolated to **control-plane only** (etcd/API server).

4. Use static Pod + logs to debug API and etcd (section 6).

---

## 8. Essential Command Cheat Sheet (Bottom-Up)

```bash
# Host layer
uptime; free -m; df -h; lsblk
ip addr; ip route
sudo systemctl status kubelet containerd
sudo journalctl -u kubelet -f

# CRI layer (containerd + crictl)
sudo crictl info
sudo crictl pods
sudo crictl ps -a
sudo crictl inspect <container-id>
sudo crictl logs <container-id>
sudo ctr -n k8s.io c ls

# Kubelet layer
sudo cat /var/lib/kubelet/config.yaml
sudo cat /etc/kubernetes/kubelet.conf
sudo journalctl -u kubelet | tail

# Network
ip addr; ip route
sudo iptables -t nat -L KUBE-SERVICES -n
sudo ipvsadm -Ln

# Control plane
kubectl get nodes
kubectl get pods -n kube-system
kubectl cluster-info
kubectl cluster-info dump

# Node debug via kubectl
kubectl debug node/<node> -it --image=ubuntu
```

---

## 9. When to Stop Going Downwards

Reverse‑engineering is powerful, but know when **not** to go deeper:

- If `kubectl describe` and `kubectl logs` clearly show an **app bug**, fix code/config first.
- If all nodes & kubelet & runtime are fine, but rollout/scale decisions are wrong → look at **controllers and higher-level policies**.
- If multiple clusters show the same issue → suspect **CI/CD or GitOps** layers.

Use this guide when you suspect that:

- Kubelet, CRI, or network/storage stack is misbehaving.
- Control-plane is partially broken but workloads are still running.
- `kubectl` output conflicts with what you expect on the host.

In those cases, **start from the node and runtime, then climb back up**.
