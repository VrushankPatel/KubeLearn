**A. 10 second summary of Kubernetes**  
Kubernetes (K8s) is an open-source system that automates deploying, scaling, and operating containerized applications. You declare what you want (YAML), and the control plane makes it real across any number of machines while self-healing failures. Think of it as the OS for your entire fleet of containers. (Official Kubernetes homepage, v1.35, April 2026)

**B. Main concepts grouped by category**  
I explain from cluster level down to pod level, exactly as requested. Every concept follows the exact format you asked for.

### 1. Cluster Fundamentals & Control Plane
**Cluster**  
- **One-line definition**: A set of worker machines (Nodes) controlled by a control plane.  
- **Why it exists**: To run your apps across many machines with automatic failover, scaling, and networking.  
- **Real-world production example**: A bank runs 500+ nodes split across 3 availability zones so if one zone dies, workloads move automatically.  
- **kubectl example**: `kubectl get nodes`  
- **Interview trap**: "A cluster is just a bunch of VMs" → No. It has a control plane that owns the desired state.

**Control Plane Components** (API Server, etcd, Controller Manager, Scheduler, kubelet)  
- **One-line definition**: The brain of the cluster (API Server + etcd + controllers + scheduler) + the kubelet agent on every node.  
- **Why it exists**: API Server is the only entry point; etcd stores all state; controllers reconcile reality to your YAML; scheduler places Pods; kubelet actually runs them.  
- **Real-world production example**: In a 10,000-node cluster at a hyperscaler, etcd is run on dedicated machines with 3–5 replicas for HA.  
- **kubectl example**: `kubectl get componentstatuses` (legacy) or check control-plane Pods in `kube-system`.  
- **Interview trap**: Confusing kubelet (node agent) with Scheduler (control-plane only).

**Node** (and NodePool – cloud-specific)  
- **One-line definition**: A physical or virtual machine that runs Pods.  
- **Why it exists**: To provide CPU/memory/storage for Pods.  
- **Real-world production example**: Google Kubernetes Engine uses NodePools to group nodes by machine type (e.g., one for CPU-heavy ML, one for general web).  
- **kubectl example**: `kubectl describe node <name>`  
- **Interview trap**: NodePool is NOT core Kubernetes – it is GKE/AKS/EKS specific.

### 2. Pod & Containers
**Pod**  
- **One-line definition**: The smallest deployable unit – one or more containers that share network, storage, and lifecycle.  
- **Why it exists**: Containers inside a Pod are tightly coupled (e.g., main app + helper) and must be co-located on the same machine.  
- **Real-world production example**: A payment service Pod runs the Go app + a sidecar that ships metrics to Prometheus.  
- **kubectl example**: `kubectl get pods -o wide`  
- **Interview trap**: "Pod = container" → Wrong. One Pod can have multiple containers.

**Container** (inside Pod)  
- **One-line definition**: The Docker/OCI runtime unit that actually runs your app code.  
- **Why it exists**: Isolates your app from the host and other apps.  
- **Real-world production example**: Every microservice at Spotify runs in its own container.  
- **YAML snippet** (in Pod spec): `containers: - name: myapp image: myapp:1.2.3`

**Init Container**  
- **One-line definition**: Special containers that run to completion before main containers start.  
- **Why it exists**: To do setup tasks (migrate DB, download secrets, wait for dependencies).  
- **Real-world production example**: E-commerce checkout Pod runs an init container that runs DB schema migration before the Java app starts.  
- **YAML example**:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: init-demo
spec:
  initContainers:
  - name: init-myservice
    image: busybox
    command: ['sh', '-c', 'until nslookup myservice; do echo waiting; sleep 2; done']
  containers:
  - name: myapp
    image: nginx
```
- **Interview trap**: People think init containers run forever → No, they must exit 0.

**Sidecar Container**  
- **One-line definition**: A helper container that runs alongside the main container in the same Pod (stable since v1.33).  
- **Why it exists**: To add cross-cutting concerns without changing your main app (logging, mesh, file sync).  
- **Real-world production example**: Istio injects an Envoy sidecar into every Pod for zero-trust networking.  
- **YAML**: Add a second container with `restartPolicy: Always`.  
- **Interview trap**: "Sidecar is the same as init" → Init runs once; sidecar runs the whole Pod lifetime.

### 3. Workload Controllers
**ReplicaSet**  
- **One-line definition**: Ensures a fixed number of Pod replicas run.  
- **Why it exists**: Basic scaling and self-healing (replaces failed Pods).  
- **Real-world**: Rarely used directly today.  
- **Trap**: Almost always managed by Deployment.

**Deployment**  
- **One-line definition**: Manages ReplicaSets to do rolling updates, scaling, and rollbacks for stateless apps.  
- **Why it exists**: Declarative way to update Pods safely.  
- **Real-world production example**: Netflix deploys thousands of stateless microservices with zero-downtime rolling updates.  
- **YAML example** (realistic):
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  replicas: 10
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        resources:
          requests:
            cpu: "500m"
            memory: "512Mi"
```
- **kubectl**: `kubectl rollout status deployment/frontend`  
- **Interview trap**: "Deployment creates Pods directly" → No, it creates ReplicaSet → Pods.

**StatefulSet** (compare below)  
- **One-line definition**: Manages stateful Pods with stable identity, ordering, and storage.  
- **Why it exists**: Databases need persistent hostname and ordered startup.  
- **Real-world production example**: MongoDB or Cassandra clusters at Uber – each Pod gets its own PersistentVolume and DNS name like `mongo-0`.  
- **YAML**: Similar to Deployment but adds `volumeClaimTemplates` and `serviceName`.  
- **Trap**: People use it for stateless apps → waste of complexity.

**DaemonSet**  
- **One-line definition**: Runs exactly one Pod on every (or selected) Node.  
- **Why it exists**: Node-level agents (logging, monitoring, networking).  
- **Real-world production example**: Fluentd or Filebeat on every node to ship logs to Elasticsearch.  
- **Trap**: "DaemonSet scales with replicas" → No, it ignores replicas field.

**Job & CronJob**  
- **One-line definition**: Job runs a Pod to completion; CronJob runs it on a schedule.  
- **Why it exists**: Batch processing and scheduled tasks.  
- **Real-world**: Nightly ETL job that processes 10 TB of sales data.  
- **YAML**: Job has `completions` and `backoffLimit`.  
- **Trap**: "Job = one Pod forever" → No, it stops when done.

### 4. Networking & Service Discovery
**Service**  
- **One-line definition**: Stable virtual IP + DNS name that load-balances to Pods.  
- **Why it exists**: Pods die and get new IPs; Services give stable endpoint.  

**Service types** (side-by-side later):  
- **ClusterIP**: Internal only (default).  
- **NodePort**: Exposes on every Node’s IP + high port.  
- **LoadBalancer**: Provisions cloud LB (AWS NLB, GCP, etc.).  
- **Headless**: No VIP – returns Pod IPs directly (for StatefulSets).  

**Ingress & Ingress Controller**  
- **One-line definition**: HTTP(S) router that sits in front of multiple Services.  
- **Why it exists**: One external LB for many apps with path/host routing and TLS termination.  
- **Real-world**: All production web apps use NGINX or Traefik Ingress Controller.  
- **Note 2026**: Ingress API is frozen; Kubernetes now recommends Gateway API.  
- **YAML** (above in tool result).  
- **Trap**: "Ingress is a load balancer" → No, it is a set of rules; the Controller implements it.

**Service Mesh** (Istio, Linkerd)  
- **One-line definition**: Extra layer that adds mTLS, traffic management, observability without changing app code (usually via sidecars).  
- **Why it exists**: Microservices need zero-trust security and canary routing.  
- **Real-world**: Istio at Adidas for 1000+ services.  
- **Trap**: "Service Mesh replaces Kubernetes networking" → It builds on top.

**DNS & CoreDNS / kube-proxy**  
- **CoreDNS**: Cluster DNS server that resolves Service and Pod names.  
- **kube-proxy**: Runs on every node and implements Service VIPs (iptables or IPVS mode).  
- **Trap**: "DNS is magic" → It is just CoreDNS watching the API.

### 5. Configuration
**Environment Variables & Downward API**  
- **Env vars**: Simple key-value inside container.  
- **Downward API**: Injects Pod metadata (name, IP, labels, node name) as env vars or files.  
- **Why**: Apps need runtime info without hard-coding.  
- **Trap**: Secrets in env vars appear in `kubectl describe` → use mounted Secrets instead for production.

### 6. Storage
**Volume / PersistentVolume / PersistentVolumeClaim / StorageClass / CSI / Ephemeral Volume**  
- **Volume**: Directory mounted into Pod (ephemeral by default).  
- **PersistentVolume (PV)**: Cluster-level storage resource that survives Pod deletion.  
- **PersistentVolumeClaim (PVC)**: User request for storage.  
- **StorageClass**: Template for dynamic PV creation.  
- **CSI**: Standard way to plug in cloud or on-prem storage (preferred since ~1.21).  
- **Ephemeral Volume**: Tied to Pod lifecycle (emptyDir, etc.).  
- **Why**: Apps need persistent data.  
- **Real-world**: EKS uses EBS CSI driver; databases claim 100 Gi PVCs.  
- **YAML examples** (see tool results above).  
- **Trap**: "PVC = PV" → PVC is the claim; PV is the actual storage.

### 7. Scheduling & Resources
**Scheduler / Taints & Tolerations / Affinity & Anti-affinity**  
- **Scheduler**: Places Pods on Nodes that have capacity and satisfy rules.  
- **Taints/Tolerations**: "No schedule" or "No execute" marks on Nodes.  
- **Affinity**: "Put this Pod near/away from that Pod/Node".  
- **Real-world**: Dedicated GPU NodePool with taint; only ML Pods have toleration.  

**Resource Requests & Limits / QoS Classes**  
- **Requests**: Minimum guaranteed resources for scheduling.  
- **Limits**: Max a container can use (CPU throttled, memory → OOMKill).  
- **QoS**: Guaranteed (requests=limits), Burstable, BestEffort.  
- **Trap**: Setting only limits → Pods get BestEffort QoS and can be killed first.

**HPA / VPA / Cluster Autoscaler** (compare in C)

### 8. Security & Isolation
**Namespace / RBAC (Role, ClusterRole, RoleBinding, ClusterRoleBinding, ServiceAccount) / Pod Security Standards / NetworkPolicy / Admission Controller / OPA Gatekeeper**  
- **Namespace**: Virtual cluster inside a cluster (resource isolation).  
- **RBAC**: Who can do what on which resources.  
- **Pod Security Standards**: Enforced admission (Privileged, Baseline, Restricted).  
- **NetworkPolicy**: Firewall rules between Pods.  
- **Admission Controller / OPA Gatekeeper**: Policy engine that rejects bad YAML before creation.  
- **Real-world**: Banks put each team in its own Namespace + strict NetworkPolicy.  
- **Trap**: "ClusterRole is cluster-wide" → Yes; Role is namespace-scoped.

### 9. Observability & Troubleshooting
**Probes (Liveness, Readiness, Startup)**  
- **Liveness**: If fails → restart container.  
- **Readiness**: If fails → remove from Service endpoints.  
- **Startup**: Gives slow apps time before liveness/readiness kick in (newer).  
- **Real-world**: Slow Java Spring Boot uses startup probe for 60s.  
- **Trap**: Using liveness for readiness → kills healthy Pods.

**kubectl describe / logs** + common Pod phases and errors (see D).

**Logs, Events, Metrics Server, Prometheus, Grafana**  
Standard observability stack.

### 10. Packaging & Advanced Ops
**Helm / Kustomize** – package managers.  
**Operators / CRD** – custom controllers for complex apps (e.g., Postgres Operator).  
**GitOps (Argo CD, Flux)** – declarative Git → cluster sync.  
**Deployment strategies**: Rolling Update (default), Blue-Green, Canary.  
**Immutable Infrastructure / IaC / CI/CD Pipeline** – standard modern practice.

**C. Side-by-side comparisons**

| Concept                  | Deployment                  | StatefulSet                          | DaemonSet                     |
|--------------------------|-----------------------------|--------------------------------------|-------------------------------|
| Use case                 | Stateless apps              | Stateful (DBs, queues)               | Node agents                   |
| Pod identity             | Random                      | Stable (pod-0, pod-1)                | One per Node                  |
| Storage                  | Shared or ephemeral         | volumeClaimTemplates (unique PV)     | Usually none                  |
| Ordering                 | No                          | Yes (0→1→2)                          | No                            |
| Scaling                  | replicas field              | replicas field                       | Ignores replicas              |

**Service types**  
- ClusterIP: internal only  
- NodePort: external via Node IP (dev/test)  
- LoadBalancer: cloud LB (prod)  
- Headless: StatefulSet discovery (no load balancing)

**HPA vs VPA vs Cluster Autoscaler**  
- HPA: scales number of Pods (CPU/memory)  
- VPA: scales resources per Pod (vertical)  
- Cluster Autoscaler: adds/removes Nodes  

**Ingress vs Service**  
- Service: L4 (TCP/UDP)  
- Ingress: L7 (HTTP paths, hosts)  

**Probe types**  
- Startup: only at start (slow apps)  
- Readiness: traffic ready?  
- Liveness: healthy? → restart  

**Pod vs Container**  
- Container = code runtime  
- Pod = wrapper that gives shared network + storage + lifecycle

**D. Common errors and how to debug them**  
- **ErrImagePull / ImagePullBackOff**: Wrong image name/tag or private registry (no secret). `kubectl describe pod` → Events. Fix: `kubectl create secret docker-registry` + imagePullSecrets.  
- **CrashLoopBackOff**: App exits immediately. `kubectl logs <pod> --previous` + liveness probe failing. Check app startup.  
- **OOMKilled**: Memory limit too low. `kubectl describe` shows "OOMKilled". Increase limit or fix memory leak.  
- **CreateContainerConfigError**: Missing Secret/ConfigMap or bad volume mount. `kubectl describe`.  
- **RunContainerError**: Runtime issue (e.g., wrong command). Logs + describe.  
- **Pending**: No Node can schedule (resources, taints, affinity). `kubectl describe pod` → Events.  
- **ContainerCreating**: Slow image pull or volume mount. Wait or check node disk.  
- **Terminating**: Finalizers or stuck kubelet. `kubectl delete pod --grace-period=0 --force`.  
- **Evicted**: Node pressure (memory/disk). Check Node conditions.

**E. Production examples**  
- **Deployment**: Frontend of Amazon.com (stateless, auto-scaled).  
- **StatefulSet**: Kafka clusters at LinkedIn (ordered, persistent).  
- **DaemonSet**: Datadog agent on every node at Airbnb.  
- **CSI + PVC**: All production databases on EKS/GKE use cloud CSI drivers.  
- **GitOps + Argo CD**: Adidas and many banks run everything from Git.  
- **Istio**: Bloomberg and many fintechs for mTLS everywhere.

**F. Interview questions and answers** (one per major section)

1. **Fundamentals**: "What happens when you run `kubectl apply`?"  
   **Model answer**: API Server validates → stores in etcd → controllers watch and reconcile (Scheduler places, kubelet runs).

2. **Workloads**: "When would you choose StatefulSet over Deployment?"  
   **Model answer**: When Pods need stable hostname, ordered rollout, and unique PersistentVolumes (databases).

3. **Networking**: "Why use Headless Service with StatefulSet?"  
   **Model answer**: So clients can connect directly to individual Pods by DNS (e.g., mongo-0.mongo.default.svc.cluster.local).

4. **Storage**: "What is the difference between PV and PVC?"  
   **Model answer**: PV is the actual storage; PVC is the user’s claim that gets bound to a PV.

5. **Scheduling**: "How do you dedicate Nodes to a workload?"  
   **Model answer**: Taint the Nodes + toleration on Pods + NodeAffinity.

6. **Security**: "How do you prevent a Pod from running as root?"  
   **Model answer**: Pod Security Standards (Restricted) + admission controller or OPA Gatekeeper policy.

7. **Probes**: "Difference between liveness and readiness?"  
   **Model answer**: Liveness kills/restarts; readiness only removes from Service endpoints.

**G. Final cheat sheet & memory tricks**  
- **Cluster → Control Plane → Node → Pod → Container** (top-down mental model).  
- **Stateless = Deployment**, **Stateful = StatefulSet**, **Node-daemon = DaemonSet**.  
- **Service = stable IP**, **Ingress = HTTP router**.  
- **Requests = guaranteed**, **Limits = max** (set both or get BestEffort).  
- **Init = before**, **Sidecar = during**.  
- Memory trick: "Pods are whales, containers are the fish inside the pod."  
- Debugging golden command: `kubectl describe pod <name> --namespace <ns>` + `kubectl logs <pod> -c <container> --previous`.  
- Production rule: Never create Pods directly; always use a controller. Never put secrets in env vars in prod. Always use Resource requests + limits.

You are now interview-ready and production-aware. Practice applying these YAMLs in a kind cluster and you will crush any Kubernetes interview.