# Kubernetes Daily Lifecycle Commands (kubeadm + kubectl + system)

> Scope: Commands commonly used in **real-world kubeadm-based clusters** from **Day‑0 (bootstrap)** through **Day‑2 (operations, upgrades, teardown)**.
>
> Focused on:
> - `kubeadm` for cluster lifecycle[cite:42]
> - `kubectl` for API interaction[cite:41][cite:8]
> - System / runtime tools (`systemctl`, `journalctl`, `crictl`, `ip`, etc.)
>
> Not intended to list every flag; instead it groups the **commands that matter daily** with realistic usage.

---

## 0. Assumptions

- You are using **kubeadm** on Linux VMs or bare metal.
- Container runtime is **containerd** (adjust for CRI‑O/Docker if needed).
- You have `sudo` access.

---

## 1. Day‑0: OS & Runtime Preparation

### 1.1 OS Prep (per node)

| Task | Command | Notes |
|------|---------|-------|
| Update OS packages (Debian/Ubuntu) | `sudo apt update && sudo apt upgrade -y` | Keep nodes up to date (kernel, security). |
| Disable swap (runtime) | `sudo swapoff -a` | Required by kubelet; also comment swap in `/etc/fstab`. |
| Check swap is disabled | `free -m` | Swap column should show 0 used. |
| Enable IP forwarding | `echo 1 \| sudo tee /proc/sys/net/ipv4/ip_forward` | Needed for routing across pods/services. |
| Persist IP forwarding | `echo "net.ipv4.ip_forward=1" \| sudo tee /etc/sysctl.d/99-k8s.conf && sudo sysctl --system` | Load at boot. |
| Install basic tools | `sudo apt install -y curl wget vim git net-tools` | Helper tools. |
| Check hostname | `hostnamectl` | Each node should have unique, meaningful hostname. |

### 1.2 Container Runtime (containerd)

| Task | Command | Notes |
|------|---------|-------|
| Install containerd (Ubuntu example) | `sudo apt install -y containerd` | Or follow official containerd docs. |
| Generate default config | `sudo containerd config default \| sudo tee /etc/containerd/config.toml` | Baseline config. |
| Set SystemdCgroup (in config) | Edit `/etc/containerd/config.toml` → `SystemdCgroup = true` | Match kubelet cgroup driver. |
| Restart containerd | `sudo systemctl restart containerd` | Apply config changes. |
| Enable containerd at boot | `sudo systemctl enable containerd` | Run on startup. |
| Check containerd | `sudo systemctl status containerd` | Ensure `active (running)`. |

### 1.3 Install kubeadm, kubelet, kubectl

Follow the version-appropriate instructions from Kubernetes docs[cite:42], then daily commands:

| Task | Command | Notes |
|------|---------|-------|
| Check kubelet | `sudo systemctl status kubelet` | Should be installed but will often be in `CrashLoop` until `kubeadm init/join`. |
| Enable kubelet | `sudo systemctl enable kubelet` | Start at boot. |
| Check versions | `kubeadm version`, `kubectl version --client`, `kubelet --version` | Ensure versions are compatible (N‑1, N‑2). |

---

## 2. Day‑1: Cluster Initialization (Control Plane)

### 2.1 `kubeadm init` + kubeconfig

| Task | Command | Notes |
|------|---------|-------|
| Initialize first control-plane node | `sudo kubeadm init --pod-network-cidr=10.244.0.0/16` | Choose `--pod-network-cidr` based on CNI (Flannel: 10.244.0.0/16, Calico defaults, etc.)[cite:42]. |
| Use specific Kubernetes version | `sudo kubeadm init --kubernetes-version=v1.30.0 --pod-network-cidr=10.244.0.0/16` | For pinning cluster version. |
| Configure kubectl for root | `mkdir -p $HOME/.kube && sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config && sudo chown $(id -u):$(id -g) $HOME/.kube/config` | Use after `kubeadm init` to control cluster from this node. |
| Show cluster info | `kubectl cluster-info` | Verify API server and CoreDNS/kube-proxy after CNI. |
| View cluster nodes | `kubectl get nodes` | Initially only this control-plane node, often `Ready` but tainted. |

### 2.2 Install Pod Network (CNI)

Example: **Calico** (always check latest from docs):

| Task | Command | Notes |
|------|---------|-------|
| Install Calico CNI | `kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/calico.yaml` | Wait for Calico pods to be `Running` in `kube-system`. |
| Verify Calico pods | `kubectl get pods -n kube-system -l k8s-app=calico-node` | Ensure all DaemonSet pods are `Running`. |

Example: **Flannel**:

| Task | Command | Notes |
|------|---------|-------|
| Install Flannel CNI | `kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml` | CIDR must match `--pod-network-cidr`. |
| Verify Flannel pods | `kubectl get pods -n kube-system -l app=flannel` | Pods should be `Running`. |

### 2.3 Tokens & Join Command

| Task | Command | Notes |
|------|---------|-------|
| Get default join command (after init) | Shown at end of `kubeadm init` | Save for joining nodes. |
| Print join command later | `kubeadm token create --print-join-command` | Regenerates join command with new token[cite:42]. |
| List tokens | `kubeadm token list` | See existing bootstrap tokens (TTL, usage). |
| Create token with TTL | `kubeadm token create --ttl=2h` | For time-limited joins. |

---

## 3. Day‑1: Add More Control-Planes & Workers

### 3.1 Join Additional Control-Plane Nodes

Run on each **new control-plane node**:

| Task | Command | Notes |
|------|---------|-------|
| Join as control-plane | `sudo kubeadm join <API_SERVER_IP>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash> --control-plane --certificate-key <key>` | `--certificate-key` is printed by `kubeadm init` or `kubeadm init phase upload-certs --upload-certs`[cite:42]. |
| Check control-plane pods | `kubectl get pods -n kube-system -o wide` | `kube-apiserver`, `kube-controller-manager`, `kube-scheduler`, `etcd` replicated across control-plane nodes. |

### 3.2 Join Worker Nodes

On each **worker node**:

| Task | Command | Notes |
|------|---------|-------|
| Join worker | `sudo kubeadm join <API_SERVER_IP>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>` | Node registers and kubelet starts pods. |
| Check node status | `kubectl get nodes -o wide` | Workers should appear as `Ready` once kubelet and CNI are OK. |

---

## 4. Day‑1: Basic Cluster Validation

### 4.1 Core Checks

| Task | Command | Notes |
|------|---------|-------|
| List all nodes | `kubectl get nodes -o wide` | Check `STATUS`, `ROLES`, `VERSION`, `INTERNAL-IP`. |
| System pods health | `kubectl get pods -n kube-system` | `coredns`, `kube-proxy`, CNI pods all `Running`. |
| Check events | `kubectl get events -A --sort-by=.metadata.creationTimestamp` | Early signal for cluster misconfig. |

### 4.2 Test Application

| Task | Command | Notes |
|------|---------|-------|
| Deploy test nginx | `kubectl create deployment nginx --image=nginx` | Simple workload. |
| Expose test deployment | `kubectl expose deployment nginx --port=80 --type=ClusterIP` | Internal service. |
| Get pods & svc | `kubectl get pods,svc` | Confirm Pod readiness and Service creation. |
| Test from debug pod | `kubectl run curl --image=radial/busyboxplus:curl -it --rm --restart=Never -- sh` then `curl http://nginx` | Verify service + DNS resolution. |

---

## 5. Day‑1 / Day‑2: Everyday `kubectl` for Ops

### 5.1 Namespaces & Contexts

| Task | Command | Notes |
|------|---------|-------|
| List namespaces | `kubectl get ns` | Logical boundaries for teams/apps. |
| Set default namespace | `kubectl config set-context --current --namespace=<ns>` | Avoid `-n` each time. |
| List contexts | `kubectl config get-contexts` | Inspect configured clusters. |
| Switch context | `kubectl config use-context <name>` | Move between dev/stage/prod. |

### 5.2 Workloads

| Task | Command | Notes |
|------|---------|-------|
| List pods | `kubectl get pods [-n <ns>]` | Use `-o wide` for node/IP info. |
| Describe pod | `kubectl describe pod <name> -n <ns>` | Deep debug (events, containers, probes). |
| List deployments | `kubectl get deploy [-n <ns>]` | See `READY` status. |
| Scale deployment | `kubectl scale deploy <name> --replicas=3 -n <ns>` | Manual scale outside HPA. |
| Rollout status | `kubectl rollout status deploy/<name> -n <ns>` | Wait for rollout completion. |
| Rollout history | `kubectl rollout history deploy/<name> -n <ns>` | See previous revisions. |
| Rollback deployment | `kubectl rollout undo deploy/<name> -n <ns>` | Revert to previous revision. |
| Restart deployment | `kubectl rollout restart deploy/<name> -n <ns>` | Trigger restart to pick new config. |
| Delete deployment | `kubectl delete deploy <name> -n <ns>` | Clean up workload. |

### 5.3 Logs & Debug

| Task | Command | Notes |
|------|---------|-------|
| Logs of pod | `kubectl logs <pod> [-c <container>] -n <ns>` | General debugging. |
| Stream logs | `kubectl logs -f <pod> -n <ns>` | Tail logs during incidents. |
| Logs of previous instance | `kubectl logs <pod> --previous -n <ns>` | For CrashLoopBackOff analysis. |
| Exec into container | `kubectl exec -it <pod> -n <ns> -- /bin/sh` | Use ephemeral containers for hardened images. |
| Ephemeral debug container | `kubectl debug <pod> -it --image=nicolaka/netshoot -n <ns>` | Attach tools into Pod’s namespace. |
| Check events | `kubectl get events -n <ns> --sort-by=.metadata.creationTimestamp` | First go-to for scheduling/pull issues. |

### 5.4 Services, Ingress, Networking

| Task | Command | Notes |
|------|---------|-------|
| List services | `kubectl get svc -n <ns>` | Check `CLUSTER-IP`, `EXTERNAL-IP`, `PORT(S)`. |
| Describe service | `kubectl describe svc <name> -n <ns>` | Validate selectors, ports, endpoints. |
| List EndpointSlices | `kubectl get endpointslices -n <ns>` | Check backend Pods for services. |
| List ingress | `kubectl get ingress -n <ns>` | Ingress routes & hosts. |
| Describe ingress | `kubectl describe ingress <name> -n <ns>` | Check rules, TLS, backend services. |
| Port-forward to pod | `kubectl port-forward pod/<name> 8080:80 -n <ns>` | Local testing without LB. |
| Port-forward to service | `kubectl port-forward svc/<name> 8080:80 -n <ns>` | Test service chain locally. |

---

## 6. Day‑2: Node Management & Maintenance

### 6.1 Node Status & Labels

| Task | Command | Notes |
|------|---------|-------|
| List nodes | `kubectl get nodes -o wide` | See IPs, roles, version, OS. |
| Describe node | `kubectl describe node <name>` | Conditions, capacity, taints, labels. |
| Label node | `kubectl label node <name> disktype=ssd` | For nodeSelector/affinity. |
| Remove label | `kubectl label node <name> disktype-` | Note trailing `-` to remove. |
| Taint node | `kubectl taint nodes <name> key=value:NoSchedule` | Reserve node for specific workloads. |
| Remove taint | `kubectl taint nodes <name> key=value:NoSchedule-` | Allow general scheduling again. |

### 6.2 Cordon, Drain, Uncordon

| Task | Command | Notes |
|------|---------|-------|
| Cordon node | `kubectl cordon <node>` | Mark unschedulable (no new pods). |
| Drain node | `kubectl drain <node> --ignore-daemonsets --delete-emptydir-data` | Evict Pods before maintenance; respect PDBs. |
| Uncordon node | `kubectl uncordon <node>` | Allow scheduling after maintenance. |

### 6.3 Systemd & Logs (per node)

| Task | Command | Notes |
|------|---------|-------|
| Check kubelet service | `sudo systemctl status kubelet` | Ensure `active (running)`. |
| Restart kubelet | `sudo systemctl restart kubelet` | After config changes. |
| View kubelet logs | `sudo journalctl -u kubelet -f` | Tail logs live. |
| Check containerd service | `sudo systemctl status containerd` | Runtime health. |
| Restart containerd | `sudo systemctl restart containerd` | For runtime issues. |
| Tail system logs | `sudo journalctl -xe` | General OS-level debugging. |

---

## 7. Day‑2: Storage & Volumes

### 7.1 PV / PVC

| Task | Command | Notes |
|------|---------|-------|
| List PVCs | `kubectl get pvc -n <ns>` | Check `STATUS` (`Bound`, `Pending`). |
| Describe PVC | `kubectl describe pvc <name> -n <ns>` | Events show binding/provision errors. |
| List PVs | `kubectl get pv` | Check `STATUS`, `CLAIM`, `STORAGECLASS`. |
| Describe PV | `kubectl describe pv <name>` | Backend details, reclaim policy. |
| List StorageClasses | `kubectl get storageclass` | Provisioning backends. |

### 7.2 Node-level Storage (Linux)

| Task | Command | Notes |
|------|---------|-------|
| List block devices | `lsblk` | See disks, partitions, mount points. |
| Show mounted filesystems | `mount \| grep -E '/var/lib/(kubelet\|containerd)'` | Validate storage paths. |
| Disk usage | `df -h` | Ensure no node disk is full (critical for etcd/control-plane). |

---

## 8. Day‑2: Cluster Upgrades (kubeadm)

> Always refer to version-specific upgrade docs; commands below reflect the common pattern[cite:42].

### 8.1 Plan Upgrade (Control Plane)

On a **control-plane node**:

| Task | Command | Notes |
|------|---------|-------|
| See current & available versions | `sudo kubeadm upgrade plan` | Shows target versions and notes. |
| Apply upgrade to control-plane | `sudo kubeadm upgrade apply v1.30.0` | Upgrades API server, controller-manager, scheduler, etcd as needed. |
| Verify control-plane pods | `kubectl get pods -n kube-system -o wide` | Check new image tags/versions. |
| Upgrade kubelet & kubectl (node OS) | `sudo apt-get install kubelet=... kubectl=...` | Pin versions according to docs, then `sudo systemctl restart kubelet`. |

### 8.2 Upgrade Worker Nodes

Per worker node:

| Task | Command | Notes |
|------|---------|-------|
| Cordon & drain | `kubectl cordon <node>`; `kubectl drain <node> --ignore-daemonsets --delete-emptydir-data` | Safely evict workloads. |
| Upgrade kubeadm | `sudo apt-get install kubeadm=...` | Prepare node. |
| Upgrade node config | `sudo kubeadm upgrade node` | Refreshes kubelet config for new control-plane version. |
| Upgrade kubelet | `sudo apt-get install kubelet=... && sudo systemctl restart kubelet` | Follow version skew rules. |
| Uncordon node | `kubectl uncordon <node>` | Put node back into rotation. |

---

## 9. Day‑2: Backup, Restore, and Teardown

### 9.1 etcd Backup (Stacked etcd via kubeadm)

On a **control-plane node** running etcd as static pod:

| Task | Command | Notes |
|------|---------|-------|
| Find etcd pod | `kubectl get pods -n kube-system -l component=etcd -o wide` | Identify controlling node. |
| Exec etcdctl | `kubectl exec -n kube-system <etcd-pod> -- etcdctl --endpoints=https://127.0.0.1:2379 --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/server.crt --key=/etc/kubernetes/pki/etcd/server.key endpoint health` | Check etcd health. |
| Snapshot etcd | `kubectl exec -n kube-system <etcd-pod> -- etcdctl --endpoints=https://127.0.0.1:2379 --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/server.crt --key=/etc/kubernetes/pki/etcd/server.key snapshot save /var/lib/etcd/snapshot.db` | Save snapshot to disk. |
| Copy snapshot off node | `kubectl cp kube-system/<etcd-pod>:/var/lib/etcd/snapshot.db ./snapshot.db` | Copy to local or backup store. |

### 9.2 Cluster Teardown (Destroy)

> Use with extreme care; this removes cluster state.

#### 9.2.1 Remove Nodes from Cluster

On a **control-plane**:

| Task | Command | Notes |
|------|---------|-------|
| Drain node | `kubectl drain <node> --ignore-daemonsets --delete-emptydir-data` | Evict workloads. |
| Delete node object | `kubectl delete node <node>` | Remove node from cluster. |

Then on the **node itself**:

| Task | Command | Notes |
|------|---------|-------|
| Reset kubeadm | `sudo kubeadm reset` | Removes kubeadm-managed files, clears iptables rules[cite:42]. |
| Remove CNI configs | `sudo rm -rf /etc/cni/net.d` | Clean network plugins. |
| Remove iptables rules (if needed) | `sudo iptables -F && sudo iptables -t nat -F && sudo iptables -t mangle -F && sudo iptables -X` | Wipe leftover K8s rules (be careful on shared hosts). |
| Stop kubelet & containerd | `sudo systemctl stop kubelet containerd` | Shutdown k8s processes. |

#### 9.2.2 Remove Whole Cluster

On **all nodes** (workers + control-planes):

1. Drain & delete nodes from cluster (from a surviving control-plane or admin host).
2. `sudo kubeadm reset` on each node.
3. Clear CNI and iptables rules.
4. Stop/disable kubelet and containerd if these hosts will be repurposed.

---

## 10. Misc Useful System & Runtime Commands

### 10.1 Container Runtime (containerd + crictl)

| Task | Command | Notes |
|------|---------|-------|
| List pods (CRI view) | `sudo crictl pods` | Lower-level than kubectl, from node’s view. |
| List containers | `sudo crictl ps -a` | All containers on the node. |
| Logs of container | `sudo crictl logs <container-id>` | When `kubectl logs` fails. |
| Pull image | `sudo crictl pull <image>` | Pre-pull large images on nodes. |

### 10.2 Networking

| Task | Command | Notes |
|------|---------|-------|
| IP addresses | `ip addr` | Check pod network interfaces on node. |
| Routes | `ip route` | Ensure pod/service CIDRs are routed properly. |
| iptables NAT table | `sudo iptables -t nat -L -n` | Inspect kube-proxy rules. |
| IPVS table | `sudo ipvsadm -Ln` | For IPVS-mode kube-proxy. |

---

## 11. References for “All Commands”

For complete, auto-generated CLI references (every subcommand & flag):

- `kubectl` full reference: https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands[cite:41]  
- `kubectl` quick reference (common commands & options): https://kubernetes.io/docs/reference/kubectl/quick-reference/[cite:8]  
- `kubeadm` reference & guides: https://kubernetes.io/docs/reference/setup-tools/kubeadm/[cite:42]  
- `kubelet` command-line reference: https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet/[cite:35]

Use `--help` on each binary for your exact installed version:

```bash
kubeadm --help
kubectl --help
kubelet --help
