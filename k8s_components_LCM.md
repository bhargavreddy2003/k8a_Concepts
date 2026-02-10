# Kubernetes Components, Roles, and Lifecycle (kubeadm → Netflix-scale)

> Goal: Understand what **every core component and lifecycle tool** does, how they interact, and how their **role evolves from Day‑0 lab cluster → production at Netflix scale**.
>
> Focused on: **kubeadm**, **kube-apiserver**, **etcd**, **kube-scheduler**, **kube-controller-manager**, **cloud-controller-manager**, **kubelet**, **kube-proxy**, **CNI**, **CoreDNS**, plus addons and ecosystem tooling.[web:10][web:47][web:56]

---

## 0. Big Picture: Control Plane vs Data Plane

- **Control Plane (brain)**:
  - `kube-apiserver`
  - `etcd`
  - `kube-scheduler`
  - `kube-controller-manager`
  - `cloud-controller-manager` (if using cloud)[web:10][web:45]
- **Data Plane / Nodes (muscle)**:
  - `kubelet`
  - `kube-proxy`
  - Container runtime (containerd/CRI-O)
  - CNI plugin (Calico/Cilium/Flannel/etc.)[web:7][web:56]

**Lifecycle tools**:
- `kubeadm` – bootstrap, upgrade, manage certs/tokens.[web:51][web:54]
- `kubectl` – CLI client to the API server for everything (deploy, debug, admin).[web:41]
- Addons (CoreDNS, metrics-server, Ingress controller, etc.) – observability, ingress, autoscaling.

---

## 1. kubeadm – Cluster Lifecycle Orchestrator

### 1.1 Role

**What it is**: A tool to bootstrap and manage “conformant” Kubernetes clusters with minimal manual glue.[web:51][web:54]

**Main responsibilities**:
- Day‑0: Initialize control plane (`kubeadm init`).
- Day‑1: Join control-plane and worker nodes (`kubeadm join`).
- Day‑2: Perform control-plane and node upgrades (`kubeadm upgrade`).
- Manage bootstrap tokens, kubelet Bootstrap TLS, and basic cert lifecycle (`upload-certs`, `renew` in recent versions).[web:42][web:54]

### 1.2 Internals

On `kubeadm init`, it:

- Runs **preflight checks** (swap off, ports, iptables/sysctl expectations).
- Generates **PKI** under `/etc/kubernetes/pki`:
  - CA, front-proxy CA, API server, etcd server/peer/client, service-account keys.[web:54]
- Writes **kubeconfig files** for:
  - `admin`, `kubelet`, `controller-manager`, `scheduler`.
- Drops **static Pod manifests** under `/etc/kubernetes/manifests` for:
  - `kube-apiserver`, `kube-controller-manager`, `kube-scheduler`, `etcd` (if “stacked” etcd topology).
  - The kubelet automatically starts these Pods.[web:54]
- Creates **bootstrap tokens** in the cluster for node joining and uploads config via ConfigMaps.[web:51]

At **Netflix-scale** you typically don’t run kubeadm manually, but:
- You **embed kubeadm** in automation (Terraform/Ansible/IaC pipelines) or use it in image baking.
- Or rely on managed control planes (EKS/GKE/AKS) but **mirror kubeadm’s logic** for virth tests and non-prod clusters.[web:48][web:57]

### 1.3 Use Cases Across Maturity

- **Scratch/lab**: Run `kubeadm init` and `kubeadm join` directly on VMs.
- **Mid-size org**: Use kubeadm via Ansible/Terraform modules to create HA control-planes, define upgrade runbooks around `kubeadm upgrade`.
- **Netflix-scale**:
  - Use kubeadm logic in **cluster factory pipelines**.
  - Template entire cluster definitions (cluster API / GitOps) and treat Kubernetes clusters as cattle.

---

## 2. kube-apiserver – The Front Door / Truth Source

### 2.1 Role

**What it is**: The **only** component that directly writes to `etcd`; exposes the Kubernetes HTTP API.[web:10][web:47]

Responsibilities:

- Handles **all API calls**: `kubectl`, kubelets, controllers, operators, CI/CD, etc.
- Enforces **authentication** (certs, tokens, OIDC), **authorization** (RBAC, Node authorizer), and **admission** (mutating/validating webhooks).
- Validates objects, applies defaults, and persists them into `etcd`. [web:10][web:45]

### 2.2 Internals

- Serves versioned API groups (`core/v1`, `apps/v1`, `batch/v1`, etc.).
- Uses **watch** and **list+watch** semantics for controllers and clients.
- Converts between API versions via **conversion webhooks** for CRDs.

At **Netflix-scale**:
- It’s fronted by an internal **L4 or L7 load balancer**, with 3–5 API server replicas across AZs.
- SLOs exist for API availability and latency; heavy watchers (e.g., operators) are tuned and limited by design.

---

## 3. etcd – Persistent Cluster State

### 3.1 Role

**What it is**: A **strongly consistent** key-value store, holding all Kubernetes state (objects) as JSON.[web:10][web:56]

Responsibilities:

- Store the entire desired/observed state:
  - Pods, Deployments, Namespaces, CRDs, ConfigMaps, Secrets, Node objects, etc.
- Support revisioned, linearizable reads/writes so that control-plane behavior is deterministic.

### 3.2 Internals

- Implements the **Raft consensus algorithm** for leader election and log replication.
- Each write goes through leader; followers replicate log.
- Data directory (kubeadm default) under `/var/lib/etcd`; snapshots via `etcdctl`.[web:54]

At **Netflix-scale**:
- etcd is treated like a **Tier‑0** system:
  - Dedicated nodes or carefully isolated workloads.
  - Strict resource reservations and IOPS/latency budgets.
  - Regular snapshots and DR procedures; multi-region replication or CR-style failover.

---

## 4. kube-scheduler – Placement Brain

### 4.1 Role

**What it is**: The scheduling process that assigns Pods to Nodes.[web:10][web:50]

Responsibilities:

- Watches for Pods without `spec.nodeName`.
- Applies scheduling **filters** (predicates) and **scoring** plugins:
  - Resource fit, nodeSelector/affinity, taints/tolerations, topology spread, etc.
- Issues **Bind** operations to set `spec.nodeName`.

### 4.2 Internals

- Plugin framework (filter, score, reserve, bind, pre/post hooks).
- Can be replaced/augmented by **custom schedulers** for specific workloads.

At **Netflix-scale**:
- Possibly multiple schedulers for:
  - **Latency-sensitive** services vs **batch** workloads.
  - Custom algorithms for bin-packing, colocation or isolation.
- Fine-tuned Pod specs and PodTopologySpreadConstraints to balance across zones.

---

## 5. kube-controller-manager – Reconciliation Engine

### 5.1 Role

**What it is**: A binary that **runs many controllers** as separate control loops.[web:10][web:53]

Key controllers:

- Deployment, ReplicaSet, StatefulSet, DaemonSet controllers.
- Job/CronJob controllers.
- Node controller (NotReady, eviction).
- Endpoints / EndpointSlice controller.
- Namespace, ServiceAccount, Token, ResourceQuota, HorizontalPodAutoscaler, etc.[web:53]

### 5.2 Internals

Each controller:

- Watches relevant objects via API server.
- Compares **desired state** (`spec`) to **current state** (`status` & children).
- Issues changes (create/delete Pods, update status fields) until they converge.[web:53]

At **Netflix-scale**:
- Many custom **operators** built on this model:
  - Database clusters, caches, queues, internal platforms (paved paths).
- Hard SLOs on reconciliation times; heavy CRDs are segmented by namespace/cluster to control load.

---

## 6. cloud-controller-manager – Cloud Integration

### 6.1 Role

**What it is**: Runs controllers that interact with cloud provider APIs.[web:10][web:48]

Responsibilities:

- Node controller: mark Nodes NotReady when cloud VM is gone.
- Route controller: maintain cloud routing tables.
- Service controller: provision cloud load balancers for `type=LoadBalancer`.
- Volume controller: attach/detach cloud disks and integrate with PVs.

At **Netflix-scale** (or EKS/GKE/AKS):
- You rely on **cloud- or distro-specific** implementation.
- Sometimes replaced by vendor-specific CCM or integrated into managed control plane.

---

## 7. kubelet – Node Agent / Enforcer

### 7.1 Role

**What it is**: Agent running on **every node**, responsible for enforcing the PodSpec on that node.[web:47][web:50]

Responsibilities:

- Watches Pods assigned to its node via the API.
- Interacts with the container runtime via CRI (containerd/CRI-O).
- Mounts volumes (CSI), injects secrets/configmaps, sets up Pod sandboxes.
- Runs liveness/readiness/startup probes and reports Pod/Node status.[web:50]

### 7.2 Internals

- Uses node-local config file (`/var/lib/kubelet/config.yaml` or similar).[web:37]
- TLS bootstraps to the cluster using bootstrap tokens and CSRs.[web:21]
- Does periodic **node heartbeats**; failure results in Node NotReady.

At **Netflix-scale**:
- kubelet config tuned per node pool:
  - cgroup drivers, pod density, eviction thresholds, CPU/mem reservations.
- Node classes: general compute, high-memory, GPU, storage-heavy; each with different kubelet and runtime parameters.

---

## 8. kube-proxy – Service Abstraction (iptables/IPVS/eBPF)

### 8.1 Role

**What it is**: Implements **Kubernetes Service** virtual IP behavior on each node.[web:10][web:46]

Responsibilities:

- Watches Services and EndpointSlices.
- Programs NAT / LB rules using:
  - iptables (legacy but common).
  - IPVS (L4 load balancing).
  - Or is replaced/augmented by eBPF-based networking (Cilium, etc.).[web:56]

At **Netflix-scale**:
- Often replaced by **eBPF-based** dataplanes for performance and observability.
- Service mesh or sidecarless mesh becomes more important for east‑west traffic shaping.

---

## 9. CNI Plugin – Pod Networking

### 9.1 Role

**What it is**: Implements the Kubernetes **networking model** using the CNI spec.[web:56]

Responsibilities:

- Create Pod network namespaces and veth interfaces.
- Assign Pod IPs (IPAM).
- Set up routing/encapsulation (BGP, VXLAN, IP-in-IP, eBPF, etc.).
- Enforce NetworkPolicies (if feature supported).

Examples:
- **Calico** – BGP/Bird, IP-in-IP, eBPF option.
- **Cilium** – eBPF-based, L3–L7 policies, service mesh.
- **Flannel** – overlay basics (VXLAN, etc.).

At **Netflix-scale**:
- Important choices:
  - Flat vs segmented IP spaces.
  - Multi-cluster routing.
  - Cross-region traffic patterns for streaming, personalization, auth, etc.[web:49][web:52]

---

## 10. CoreDNS – Cluster DNS

### 10.1 Role

**What it is**: Addon that provides **DNS-based service discovery** for Pods.[web:10][web:46]

Responsibilities:

- Answer queries for `*.svc.cluster.local`.
- Provide per-namespace service resolution.
- Forward external DNS queries upstream (to VPC/ISP DNS).

At **Netflix-scale**:
- Possibly use **split-horizon DNS** and integrate with corporate DNS.
- Heavy caching and careful TTL choices; failover to external discovery mechanisms where needed.

---

## 11. kubectl – Client / Operator Tool

### 11.1 Role

**What it is**: General-purpose client CLI for interacting with the Kubernetes API for CRUD, debugging, and administration.[web:41][web:8]

Responsibilities:

- Declarative apply (`kubectl apply`) for GitOps or direct workflows.
- Everyday operations: get/describe/logs/exec/port-forward.
- Cluster admin: nodes, RBAC, certificates, debug.

At **Netflix-scale**:
- Used by platform/SRE teams, but application teams often get **abstractions**:
  - Internal CLIs that generate manifests.
  - GitOps flows (Argo CD/Flux) that hide direct kubectl usage in production.

---

## 12. Other Key Pieces in Lifecycle

### 12.1 Metrics & Autoscaling

- **metrics-server**:
  - Provides resource utilization (CPU/mem) metrics to API server (for `kubectl top` + HPA).[web:56]
- **HorizontalPodAutoscaler (HPA)**:
  - Controller in `kube-controller-manager` that scales Pods based on metrics.
- **Cluster Autoscaler** (external component):
  - Talks to cloud provider to scale node groups based on pending Pods or utilization.

At Netflix-scale:
- Advanced autoscaling strategies:
  - Demand prediction from streaming events.
  - Custom metrics (QPS, RPS, queue length) used with HPA or custom controllers.

### 12.2 Ingress / Gateway

- **Ingress Controller** (NGINX, HAProxy, Envoy, Traefik):
  - Watches Ingress resources, configures proxies for L7 routing.
- **Gateway API**:
  - Newer resource model for traffic (Gateways, HTTPRoutes, GRPCRoutes).

At Netflix-scale:
- Layers:
  - Global front door with anycast/GeoDNS → region-level L7 → in-cluster Ingress / Gateway.
  - Integration with API gateways, A/B testing, rate limiting, device policies.

### 12.3 Operators / CRDs

- **CustomResourceDefinition (CRD)**:
  - Extend the API with domain-specific types (e.g., `KafkaCluster`, `RedisCluster`).
- **Operators**:
  - Controllers that reconcile CRDs to real infrastructure (middleware, DBs, caches).[web:53]

At Netflix-scale:
- Operators are used heavily to encode **SRE runbooks as code**:
  - Shard management, failover, maintenance windows, safe update sequences.

---

## 13. Lifecycle Phase View: Who Does What

### 13.1 Day‑0: Infrastructure & Bootstrap

Roles by component:

- **OS / Node image**:
  - Base OS, container runtime, kubeadm, kubelet, kubectl.
- **kubeadm**:
  - Generates PKI, configs, static pod manifests.
  - Sets up first control-plane node and join tokens.[web:54]
- **kubelet**:
  - Starts static pods (control plane).
- **etcd**:
  - Becomes backend store for all cluster state.
- **kube-apiserver / controllers / scheduler**:
  - Start reconciling cluster components.
- **kube-proxy / CNI / CoreDNS**:
  - Networking and DNS for Pods/Services.

At this stage, the cluster is minimal but **fully functional**.

### 13.2 Day‑1: Adding Capacity & Basic Workloads

- **kubeadm join**:
  - Add control-plane nodes or workers.
- **kubelet**:
  - Boots new nodes, registers with API, runs Pods.
- **kube-proxy/CNI/CoreDNS**:
  - Scale out with nodes; maintain service and Pod connectivity.
- **kubectl & controllers**:
  - Deploy user workloads, orchestrate rollouts, scale-out.

### 13.3 Day‑2: Operations, Upgrades, and SRE

- **kubeadm upgrade**:
  - Orchestrate control-plane and node upgrades with defined version skew rules.[web:42]
- **kube-controller-manager & custom controllers**:
  - Ongoing reconciliation; implement advanced autoscalers and platform behaviors.
- **cloud-controller-manager & autoscalers**:
  - Integrate with cloud APIs for node scaling, LB provisioning, failover.
- **Operators/CRDs**:
  - Manage stateful services (DBs, caches) with production-grade patterns.
- **Observability stack (Prometheus, Grafana, logs, tracing)**:
  - Not “core K8s” components but critical at scale.

At **Netflix-scale**:
- Multi-cluster architecture:
  - Cluster-per-region, cluster-per-tenant or environment.
  - Control-plane SLOs, strong limitations on cluster size (e.g., 5k nodes) to keep etcd & API healthy.
- Hard separation of **platform** vs **app teams**:
  - Platform team runs kubeadm (or equivalent), etcd, control planes.
  - App teams consume higher-level abstractions, not raw cluster internals.

---

## 14. Component Cheat Table – Roles & Use Cases

| Component / Tool | Layer | Primary Role | Key Use Cases |
|------------------|-------|-------------|---------------|
| `kubeadm` | Lifecycle | Bootstrap, upgrade, manage certs/tokens | Create HA clusters, plan/apply upgrades, reset cluster, integrate with IaC.[web:51][web:54] |
| `kube-apiserver` | Control plane | API front-end, auth, admission, persistence | All CRUD, controller & kubelet interactions, policy enforcement.[web:10][web:47] |
| `etcd` | Control plane | Strongly consistent KV store | Store every object, revisioned history, snapshots for backup/restore.[web:10][web:54] |
| `kube-scheduler` | Control plane | Place Pods on Nodes | Respect resource constraints, affinity, taints, custom scheduling. |
| `kube-controller-manager` | Control plane | Reconcile desired vs actual state | Workload controllers, Node, EndpointSlice, SA, Namespace, HPA.[web:10][web:53] |
| `cloud-controller-manager` | Control plane | Cloud integrations | Node lifecycle, routes, LBs, volumes in cloud environments.[web:48] |
| `kubelet` | Node | Enforce PodSpec on node | Start/stop containers, manage volumes, run probes, report health.[web:47][web:50] |
| `kube-proxy` | Node | Implement Services | Program iptables/IPVS/eBPF for ClusterIP/NodePort/LB. |
| CNI plugin | Node | Pod networking & NetworkPolicy | Pod IP allocation, routing, overlays, network policy enforcement.[web:56] |
| CoreDNS | Addon | DNS for services/pods | `*.svc.cluster.local` resolution, upstream forwarding.[web:10] |
| metrics-server | Addon | Resource metrics | HPA inputs, `kubectl top`. |
| Ingress / Gateway controllers | Addon | North–south L7 routing | HTTP(S) entrypoints, host/path-based routing. |
| Operators & CRDs | Extensibility | Domain-specific controllers | DB clusters, queues, internal platforms “as code”.[web:53] |
| `kubectl` | CLI | Human/automation interface | Deploy, debug, admin, scripting.[web:41] |

---

## 15. How to Use This in Your Own Learning

- Treat this file as an **index** of who does what.
- For each component:
  - Re-read the official concept docs from Kubernetes.io and 1–2 deep-dive blog posts linked there.[web:10][web:45][web:53]
  - Implement a **lab scenario** where you break that component (e.g., kill kube-proxy, misconfigure CNI) and observe behavior.
- As you move toward “Netflix-level” understanding:
  - Think in terms of **SLOs, failure modes, and multi-cluster** patterns rather than only per-cluster internals.
  - Encode your ops flows into operators and GitOps pipelines rather than manual use of kubeadm/kubectl.

