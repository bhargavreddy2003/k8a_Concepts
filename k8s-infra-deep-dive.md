# Kubernetes Infrastructure Deep Dive: From Bare Metal to Debugging

> Focused on **infrastructure creation**, **low-level internals**, and **real-world debugging**, assuming strong Linux and networking background.

This guide is opinionated around **kubeadm-style clusters** and bare-metal/VM deployments, because that is where you have the most control and the most things to debug. Managed services (EKS/GKE/AKS) hide some steps, but the internal concepts are the same.

---

## 1. End-to-End Cluster Bring-Up (kubeadm)

### 1.1 Big Picture Flow

For a typical on-prem / self-managed cluster:

1. **Provision machines** (physical/VM): OS, networking, storage.
2. **Install container runtime** (containerd/CRI-O).
3. **Install kubelet + kubeadm + kubectl**.
4. **Run `kubeadm init` on first control-plane node**.
5. **Install Pod network (CNI)**.
6. **Join more control-plane nodes (optional)**.
7. **Join worker nodes**.
8. **Install addons** (CoreDNS, kube-proxy, metrics-server, Ingress controller, etc.).

At a low level, kubeadm mostly does **configuration + certificate generation + static Pod manifest creation**.

### 1.2 What `kubeadm init` Actually Does Internally

From the official reference, `kubeadm init` is a composition of ordered **phases**[cite:26]:

- `preflight`: system checks (swap disabled, ports free, IP forwarding, etc.).
- `kubelet-start`: writes `/var/lib/kubelet/config.yaml`, systemd drop-ins, and starts/restarts kubelet.
- `certs`: generates all control-plane and etcd certificates into `/etc/kubernetes/pki`:
  - `/ca`: cluster CA (signs API server, controller, scheduler, and kubelet client certs).
  - `/apiserver`: cert for the API server HTTPS endpoint.
  - `/apiserver-kubelet-client`: client cert API server uses to talk to kubelets.
  - `/front-proxy-ca` and `/front-proxy-client`: for aggregator / front-proxy.
  - `/etcd-*`: etcd server, peer, healthcheck, and apiserver-etcd client certificates.
  - `/sa`: private/public key pair for signing ServiceAccount tokens.
- `kubeconfig`: writes kubeconfigs for:
  - `admin` (your `~/.kube/config`), `controller-manager`, `scheduler`, `kubelet` bootstrap.
- `control-plane`: generates **static Pod manifests** under `/etc/kubernetes/manifests` for:
  - `kube-apiserver.yaml`, `kube-controller-manager.yaml`, `kube-scheduler.yaml`.
- `etcd`: generates local etcd static Pod manifest (if using local etcd).
- `kubelet-start` & `wait-control-plane`: ensures kubelet picks up static PODs and control-plane becomes `Ready`.
- `upload-config`: uploads kubeadm & kubelet config to ConfigMaps (`kube-system` namespace).
- `upload-certs`: stores encrypted certs in `kubeadm-certs` Secret to help join additional control-plane nodes.
- `mark-control-plane`: labels & taints the node as a control-plane node.
- `bootstrap-token`: generates tokens for joining nodes.
- `addon`: deploys CoreDNS & kube-proxy as Deployments/DaemonSets.

Kubelet watches `/etc/kubernetes/manifests`; when yaml files appear there, it launches them as Pods in the `kube-system` namespace and restarts them when files change.

### 1.3 Key `kubeadm` Infra Commands

| Use case | Command | What it changes / internals |
|---------|---------|-----------------------------|
| Initialize first control-plane node | `kubeadm init --pod-network-cidr=10.244.0.0/16` (example) | Writes certs to `/etc/kubernetes/pki`, static Pod manifests under `/etc/kubernetes/manifests`, config in `/etc/kubernetes`, starts control-plane, generates join tokens[cite:26]. |
| Show join command after init | `kubeadm token create --print-join-command` or `kubeadm init phase upload-certs --upload-certs` then `kubeadm token create` | Reads current tokens and control-plane endpoint, prints command string for `kubeadm join`. |
| Initialize additional control-plane node | `kubeadm join <endpoint>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash> --control-plane --certificate-key <key>` | Uses bootstrap token + CA hash to trust existing control-plane and downloads control-plane certs from `kubeadm-certs` Secret[cite:26]. |
| Join worker node | `kubeadm join <endpoint>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>` | Installs kubelet config pointing to control-plane, requests kubelet client cert (TLS bootstrap), node registers itself. |
| Check kubeadm configuration | `kubectl -n kube-system get configmap kubeadm-config -o yaml` | Shows `ClusterConfiguration` and `ClusterStatus` used for upgrades and future operations. |
| Plan cluster upgrade | `kubeadm upgrade plan` | Queries cluster API to determine current versions and prints safe upgrade paths and required steps. |
| Apply control-plane upgrade | `kubeadm upgrade apply v1.X.Y` | Regenerates manifests and certs as needed, updates image versions for control-plane static Pods. |
| Upgrade node configuration | `kubeadm upgrade node` | Refreshes node-local kubelet config and (optionally) kubelet version. |
| Reset node | `kubeadm reset` | Removes kubeadm-created files, stops kubelet, clears iptables rules, but leaves some directories for manual cleanup (like CNI). |

---

## 2. TLS, Certificates, and Node Bootstrap

### 2.1 Control-Plane Cert Layout

By default (kubeadm):

- `/etc/kubernetes/pki/ca.crt`, `ca.key`: cluster CA for API server and client certs.
- `apiserver.crt`, `.key`: API server serving cert, SANs include control-plane IP and DNS.
- `apiserver-kubelet-client.crt`, `.key`: used when API server talks to kubelets.
- `front-proxy-ca.*` and `front-proxy-client.*`: for aggregated APIs.
- `etcd/*.crt`, `.key`: etcd server, peer, client.
- `sa.key`, `sa.pub`: sign and verify ServiceAccount tokens.

### 2.2 Kubelet TLS Bootstrapping (Worker Join)

High-level flow (simplified from official TLS bootstrapping docs[cite:21][cite:27]):

1. Admin creates **bootstrap token** (`kubeadm` does this automatically) with RBAC bound to group `system:bootstrappers`.
2. On worker node, kubelet starts with a **bootstrap kubeconfig** that contains:
   - API server URL.
   - CA bundle.
   - Bootstrap token.
3. Kubelet sees no client cert yet, so it:
   - Connects to API server with token.
   - Submits a **CertificateSigningRequest (CSR)** for itself.
4. Controller manager’s CSR approval controller (plus RBAC) **auto‑approves** or waits for manual `kubectl certificate approve`.
5. Once approved, kubelet retrieves signed cert and writes normal kubeconfig; future connections use client certificate auth.
6. Optionally, kubelet **client cert rotation** is enabled; kubelet rotates its cert before expiry.

Key debug commands here:

| Use case | Command | What to look for |
|---------|---------|------------------|
| View pending CSRs | `kubectl get csr` | Check for `Pending` CSRs from new nodes; names like `csr-xxxxx`. |
| Inspect CSR | `kubectl describe csr <name>` | See the requested CN (often `system:node:<nodename>`) and groups; verify bootstrap group. |
| Approve CSR manually | `kubectl certificate approve <name>` | Grants node a client cert; node should become `Ready` after kubelet restarts connection[cite:21]. |
| Deny CSR | `kubectl certificate deny <name>` | Rejects untrusted node or misconfigured kubelet; node remains Unauthorized. |

If TLS bootstrap fails, you usually see:

- API server logs: auth errors for `system:bootstrap:...` or invalid token.
- Kubelet logs complaining it cannot connect or unauthorized.

---

## 3. Control-Plane Components: Runtime Behavior

### 3.1 kube-apiserver

- Binary launched from static Pod manifest under `/etc/kubernetes/manifests/kube-apiserver.yaml`.
- Talks to:
  - etcd via `--etcd-servers`, using `apiserver-etcd-client` certs.
  - kubelets via `--kubelet-client-certificate` and `--kubelet-client-key`.
- Maintains **watch connections** to etcd for clients doing long‑poll watches.
- Enforces authn, authz, and admission. Every mutation (create/update/delete) must pass all three.

Critical flags (kubeadm typically sets these):

- `--advertise-address`, `--bind-address`, `--secure-port`: how clients reach the API.
- `--authorization-mode=Node,RBAC`: node authorizer + RBAC for everything else.
- `--client-ca-file`: CA bundle for client cert validation.
- `--tls-cert-file`, `--tls-private-key-file`: serving certs.
- `--service-account-key-file`: verify ServiceAccount tokens.

**What changes when you apply a manifest?**

1. `kubectl apply` → API server validates & stores object in etcd.
2. Controllers and kubelets have watches open; they see updated object.
3. Controllers reconcile (create/delete Pods, update status). kubelet enforces PodSpec on its node.

### 3.2 kube-controller-manager

- Runs multiple controllers in one process (each in its own goroutine). Examples:
  - **DeploymentController**: ensures correct ReplicaSets exist and scaled.
  - **ReplicaSetController**: ensures right number of Pods for each RS.
  - **NodeController**: marks nodes `NotReady`, evicts Pods.
  - **ServiceController**: provisions external load balancers for `type=LoadBalancer`.
  - **EndpointSliceController**: maintains EndpointSlices for Services.

Each controller does:

- **List + Watch** relevant resources.
- Compare desired state (spec) vs actual (status/child objects).
- Issue updates to converge.

Debug tip: if reconcilation is not happening (e.g., RS not matching Deployment), check **controller logs** in the `kube-controller-manager` Pod and the **events** on the resource.

### 3.3 kube-scheduler

- Watches for Pods without `spec.nodeName`.
- Runs through plugins (filter and score) to choose node:
  - Filters: PodFitsResources, PodToleratesNodeTaints, PodMatchNodeSelector, etc.
  - Scores: LeastRequestedPriority, BalancedAllocation, topology spreading.
- Writes a **Bind** object, setting Pod `spec.nodeName`.

Debug patterns:

- Pods stuck in `Pending`: usually scheduling or PVC issues.
- Use `kubectl describe pod` → look at **Events** bottom section to see scheduler errors: `0/3 nodes are available: 1 Insufficient cpu, 2 node(s) had taint {…} that the pod didn't tolerate.`

### 3.4 kubelet

- Node agent that:
  - Watches PodSpecs for its node.
  - Talks to container runtime (CRI) to start/stop containers.
  - Mounts volumes, injects secrets/configmaps.
  - Runs liveness/readiness/startup probes.
  - Reports Node and Pod status back.

Key kubelet config knobs (often in `/var/lib/kubelet/config.yaml`):

- `cgroupDriver` (must match container runtime).
- `kubeReserved`, `systemReserved`: reserve resources for system daemons.
- `authentication` and `authorization` settings.
- `rotateCertificates` and `serverTLSBootstrap`.

To debug kubelet:

- `journalctl -u kubelet` (systemd) or process logs.
- On node: `sudo crictl ps`, `sudo crictl logs` if using containerd.

---

## 4. Pod Network / CNI Deep Dive

### 4.1 CNI Responsibilities

A CNI plugin (Calico, Cilium, Flannel, etc.) is responsible for:

- Allocating Pod IPs (often via IPAM plugin).
- Programming **routing/encapsulation** so that:
  - Pod ↔ Pod traffic across nodes works.
  - Node ↔ Pod traffic works.
- Enforcing NetworkPolicies (if supported).

General Pod creation networking flow:

1. kubelet calls CRI to create Sandbox (pod network namespace).
2. CRI calls CNI binary (e.g., `/opt/cni/bin/calico`) with `ADD` operation.
3. CNI:
   - Creates veth pair (host side + pod side), moves one end into pod namespace.
   - Assigns IP to pod-end interface.
   - Programs routes/encapsulations according to plugin (BGP, VXLAN, eBPF, etc.).

### 4.2 Service Implementation (kube-proxy / eBPF)

**kube-proxy (iptables/IPVS mode)**:

- Watches Services and EndpointSlices.
- For each Service:
  - Programs iptables rules or IPVS virtual services mapping `ClusterIP:port` to Pod IPs and ports.

Debug commands on node:

- `iptables -t nat -L -n | grep KUBE-SVC` (iptables mode) to see rules.
- `ipvsadm -Ln` (IPVS mode) to see load-balancing tables.

**eBPF-based service proxies (Cilium, etc.)**:

- Program eBPF maps in kernel to implement load-balancing instead of iptables.
- Use plugin-specific CLI (e.g., `cilium bpf lb list`).

### 4.3 Network Debug Scenarios

| Scenario | Commands | What to inspect |
|----------|----------|-----------------|
| Pod cannot reach another Pod | `kubectl exec -it <pod> -- ping <other-pod-ip>`; on nodes: `ip route`, CNI logs | Incorrect CNI config, no route to CIDR, host firewall blocking traffic. |
| Service ClusterIP not reachable | `kubectl get svc`; from node: `curl <clusterIP>:<port>`; inspect `kube-proxy` logs; `iptables -t nat -L -n | grep <svcIP>` | kube-proxy not running, wrong mode, or CNI breaking node-local iptables. |
| NetworkPolicy blocking traffic | `kubectl get networkpolicy -A`; describe relevant policies; test with `kubectl exec` + `curl`/`nc` | Policy selectors not matching as expected or default-deny in place. |

---

## 5. Storage & Volume Internals

### 5.1 PV / PVC Binding

- **PVC**: namespaced request: size, access mode, StorageClass.
- **PV**: cluster-scoped supply: actual storage with capacity, access modes, StorageClass.

Binding flow:

1. PVC is created.
2. PV controller in kube-controller-manager watches PVCs.
3. If StorageClass has **provisioner**, dynamic provisioner creates new volume then PV.
4. Controller binds PVC ↔ PV by setting `spec.volumeName` on PVC and `claimRef` on PV.

Debug commands:

| Use case | Command | Expected output |
|---------|---------|-----------------|
| List PVCs | `kubectl get pvc -n <ns>` | Status `Bound` if successful, `Pending` if waiting for PV. |
| Describe PVC | `kubectl describe pvc <name> -n <ns>` | Events show why binding failed (no matching StorageClass, provisioner errors). |
| List PVs | `kubectl get pv` | Check `STATUS`, `CAPACITY`, `STORAGECLASS`, `CLAIM`. |

If Pods are stuck in `ContainerCreating` with volume-related events, inspect:

- Node logs (kubelet) for CSI mount errors.
- CSI controller/driver logs in `kube-system` or vendor namespace.

---

## 6. Comprehensive `kubectl` View (Grouped by Actions)

> Listing literally every `kubectl` flag is not practical here; instead this is grouped by **what you are trying to do**, and covers the major verbs and flags you will use daily[cite:8].

### 6.1 CRUD / Apply / Patch

| Goal | Command | Notes |
|------|---------|-------|
| Create from file | `kubectl create -f file.yaml` | Fails if resource exists; good for initial bootstrap. |
| Declarative apply | `kubectl apply -f file-or-dir` | Create or patch resources to match files; merges under the hood. |
| Replace from file | `kubectl replace -f file.yaml` | Force full replace; fails if resource does not exist. |
| Edit live resource | `kubectl edit <kind>/<name>` | Opens editor on live object; good for quick experiments, not for GitOps. |
| Patch with JSON merge | `kubectl patch <kind>/<name> --type=merge -p '{"spec":{...}}'` | For small, targeted changes without editing the entire manifest. |
| Delete | `kubectl delete -f file.yaml` or `kubectl delete <kind> <name>` | Honors `propagationPolicy` and grace period unless overridden. |

### 6.2 Advanced Get/Output

| Goal | Command | Notes |
|------|---------|-------|
| Wide listing | `kubectl get pods -o wide` | Show node, Pod IP, restart count, etc. |
| Get as YAML/JSON | `kubectl get pod <name> -o yaml` | Use for debugging or exporting spec. |
| Filter by label | `kubectl get pods -l app=myapp` | Fundamental for multi-tenant environments. |
| JSONPath extraction | `kubectl get pod <name> -o jsonpath='{.status.podIP}'` | Scriptable extraction of fields. |
| Field selector | `kubectl get pods --field-selector=status.phase=Pending` | Server-side filtering by resource fields. |

### 6.3 Introspection & Discovery

| Goal | Command | Notes |
|------|---------|-------|
| List all resource types | `kubectl api-resources` | Shows short names, namespaced vs cluster-scoped, verbs. |
| List API versions | `kubectl api-versions` | Check for deprecated or missing API groups. |
| Explain schema | `kubectl explain deployment.spec.strategy` | Pulls from OpenAPI schema served by API server. |

### 6.4 Node Admin & Maintenance

| Goal | Command | Notes |
|------|---------|-------|
| View node status | `kubectl get nodes` / `kubectl describe node <name>` | Check `Conditions`, `Allocatable`, taints. |
| Cordon node | `kubectl cordon <node>` | Marks `unschedulable=true`; does not evict running Pods. |
| Drain node | `kubectl drain <node> --ignore-daemonsets --delete-emptydir-data` | Evicts Pods, respecting PDBs; use before maintenance or termination. |
| Uncordon | `kubectl uncordon <node>` | Allow scheduling again. |
| Label node | `kubectl label node <node> disktype=ssd` | Drives scheduling via nodeSelector/affinity. |
| Taint node | `kubectl taint nodes <node> key=value:NoSchedule` | Repels Pods except those with matching tolerations. |

### 6.5 Debug Features

| Goal | Command | Notes |
|------|---------|-------|
| Get cluster events | `kubectl get events -A --sort-by=.metadata.creationTimestamp` | One of the first commands when "something is weird". |
| Pod logs | `kubectl logs <pod> [-c <container>]` | Default gets last logs from main container. |
| Stream logs | `kubectl logs -f <pod>` | Attach for realtime monitoring during rollout. |
| Exec shell | `kubectl exec -it <pod> -- /bin/sh` | Use `bash` if available; prefer ephemeral debug containers for minimal images[cite:19]. |
| Ephemeral debug container | `kubectl debug <pod> -it --image=nicolaka/netshoot` | Mounts a debug container into existing Pod namespace for deep inspection[cite:19]. |
| Port-forward | `kubectl port-forward svc/myapp 8080:80` | Test service locally without external load balancer. |
| Resource usage | `kubectl top pods` / `kubectl top nodes` | Requires `metrics-server`; great for quick capacity checks[cite:22]. |

---

## 7. Real-World Debugging Scenarios

### 7.1 `kubectl` Cannot Talk to Cluster

Symptoms:

- `The connection to the server <host> was refused`.
- `x509: certificate signed by unknown authority`.

Checklist[cite:25]:

- Does `~/.kube/config` point to correct cluster & context?
- Is API server process running? On control-plane node, check `kubectl get pods -n kube-system` or `docker ps`/`crictl ps` for `kube-apiserver` container.
- Is firewall blocking `6443` from your workstation or bastion?
- Are client CA and server certs matching (if you recently rotated certs, ensure `kubeconfig` uses new CA bundle)?

Debug commands:

```bash
kubectl config view
kubectl config get-contexts
kubectl config current-context

# Try direct curl with the CA and token/client cert
curl https://<api-server>:6443/healthz \
  --cacert ca.crt \
  --header "Authorization: Bearer <token>"
```

### 7.2 Node Stuck `NotReady`

- On control-plane:

```bash
kubectl get nodes
kubectl describe node <node>
```

Look for:

- NetworkUnavailable, KubeletNotReady, DiskPressure, MemoryPressure conditions.

On the node:

```bash
journalctl -u kubelet
systemctl status kubelet
ip addr
ip route
```

Common causes:

- Kubelet cannot reach API server (network/firewall/DNS).
- TLS bootstrap/certificate issues.
- CNI plugin not running; `NetworkUnavailable=True`.

### 7.3 Pods Stuck in `Pending`

Use:

```bash
kubectl describe pod <name> -n <ns>
```

Check Events for:

- `0/3 nodes are available: 1 Insufficient cpu/memory.` → fix resource requests or add capacity.
- `0/3 nodes are available: 3 node(s) had taint {…} that the pod didn't tolerate.` → adjust tolerations or taints.
- `pod has unbound immediate PersistentVolumeClaims` → PVC not bound; debug PV/PVC.

### 7.4 Pods Stuck in `ContainerCreating`

Events may show:

- `Failed to pull image` → check image repo, auth, or network.
- `Failed to mount volume` → CSI driver issues.

Steps:

1. `kubectl describe pod` for detailed events.
2. Check node that Pod is assigned to: `kubectl get pod -o wide` for `NODE` field.
3. On that node, inspect kubelet logs and container runtime logs.

### 7.5 CrashLoopBackOff

Reasons:

- App crashes on startup.
- Wrong configuration/env.
- Probes misconfigured (killing container repeatedly).

Commands:

```bash
kubectl describe pod <pod>
kubectl logs <pod> --previous   # see logs from last crashed container
kubectl logs <pod> -c <container>
```

If probes are the culprit, `describe` events will show liveness/readiness failures.

### 7.6 ImagePullBackOff

Check:

- Image name/tag correctness.
- ImagePullSecrets configured on Pod or ServiceAccount.
- Registry connectivity.

```bash
kubectl describe pod <pod>
# look at "Failed to pull image" messages
```

### 7.7 Service Not Reachable

Checklist:

1. **Service exists and has endpoints**:

```bash
kubectl get svc mysvc -n <ns>
kubectl get endpointslices -l kubernetes.io/service-name=mysvc -n <ns>
```

2. **Pods Ready**: `kubectl get pods -l app=mysvc -n <ns>` → check `READY` field.
3. From another Pod in same namespace:

```bash
kubectl run tester --image=busybox -it --rm --restart=Never -- sh
/ # wget -qO- http://mysvc:80
```

4. If NodePort/LoadBalancer, test from node or external client.

If EndpointSlices are empty, Service selector does not match Pod labels.

### 7.8 DNS Resolution Issues

- Check CoreDNS Pods:

```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl logs -n kube-system <coredns-pod>
```

- From Pod:

```bash
nslookup mysvc.myns.svc.cluster.local
cat /etc/resolv.conf
```

Look for:

- Cluster DNS IP (usually `10.96.0.10` by default service CIDR) in `resolv.conf`.
- CoreDNS errors like REFUSED, SERVFAIL in logs.

---

## 8. Director-Level Infra Concerns (Low-Level Aware)

At senior director/platform lead level, the low-level details above matter because they inform:

- **Upgrade and change management**: understanding how kubeadm phases update certs and manifests lets you design safe blue/green or canary control-plane upgrades.
- **Security posture**: TLS bootstrap, certificate lifecycles, RBAC for bootstrap tokens, and node identity are central to node trust.
- **Resiliency**: OS and network choices (iptables vs eBPF, storage backends, etcd topology) map directly into RTO/RPO and SLOs.
- **Cost and performance**: Pod density, cgroup tuning, and scheduling policies affect infra spend and reliability.

The rest is less about more `kubectl` flags and more about **system design**: capacity planning, isolation strategy (multi-tenant clusters vs many clusters), operational runbooks, and incident response patterns.

---

## 9. How to Extend This for Yourself

Given the breadth of Kubernetes, no single document can literally enumerate every flag, CRD, and custom debugging story. As you build more clusters:

- Mirror this structure in your own repo (`infra/k8s-notes.md`).
- Add **distribution-specific** steps (EKS/GKE/AKS, Rancher, OpenShift) where they differ from kubeadm.
- For each incident, add **"scenario → signals → commands → fix"** sections so your future self and teammates can reuse that knowledge.
