Here is your comprehensive, brutal-truth, interview-ready guide to Kubernetes. It is built from first principles: from the cluster level (the machines and brains) down to the pod level (the applications).

---

### A. 10-Second Summary of Kubernetes
Kubernetes is an operating system for server clusters. It takes a fleet of machines and turns them into a single, massive pool of compute resources. You declare *what* you want to run, and Kubernetes figures out *how* and *where* to run it, automatically healing failures to match your desired state.

---

### B. Main Concepts Grouped by Category

#### 1. The Engine: Control Plane & Nodes
* **API Server**
    * **Definition:** The central brain and REST API front door of the cluster.
    * **Why it exists:** Validates all cluster changes and acts as the sole gateway to the datastore (etcd).
    * **Prod example:** CI/CD pipelines authenticate and send YAML manifests to the API Server.
    * **CLI:** `kubectl get apiservices`
    * **Interview Trap:** Assuming worker nodes talk to etcd. *Trap answer:* "The kubelet reads from etcd." *Correction:* Only the API server touches etcd directly.
* **etcd**
    * **Definition:** A highly available, distributed key-value store.
    * **Why it exists:** Holds the cluster's "source of truth" (the state you want vs. the current state).
    * **Prod example:** Stores the fact that you requested 5 replicas of an app; if etcd dies without backup, your cluster forgets its state.
    * **CLI:** `etcdctl snapshot save`
    * **Interview Trap:** Not knowing it's a Quorum-based system (needs an odd number of nodes, like 3 or 5, to avoid split-brain).
* **Controller Manager**
    * **Definition:** The infinite loop that watches the cluster and fixes drift.
    * **Why it exists:** If etcd says "5 pods" and the API server sees "4 pods," the Controller Manager creates the 5th.
    * **Prod example:** The Node Controller noticing a worker node went offline and triggering pod evictions.
    * **CLI:** `kubectl get componentstatuses` (deprecated, but historically used).
    * **Interview Trap:** Confusing it with the Scheduler. The Controller Manager says "we need a new pod," the Scheduler decides *where* it goes.
* **Scheduler**
    * **Definition:** The matchmaker for Pods and Nodes.
    * **Why it exists:** Finds the best server to run a newly requested pod based on CPU/RAM, rules, and constraints.
    * **Prod example:** Ensuring a heavy database pod lands on a high-memory node.
    * **CLI:** `kubectl get pods -n kube-system | grep scheduler`
    * **Interview Trap:** Believing the Scheduler *starts* the pod. It only assigns the Node name to the Pod spec; the local *kubelet* actually starts it.
* **Node**
    * **Definition:** A physical or virtual machine acting as a worker in the cluster.
    * **Why it exists:** Provides the actual CPU, RAM, and disk to run your containers.
    * **Prod example:** An AWS EC2 `m5.large` instance registered to the cluster.
    * **CLI:** `kubectl get nodes`
    * **Interview Trap:** Thinking Kubernetes creates Nodes out of thin air. K8s *manages* Nodes, but a cloud provider (via autoscalers) provisions the actual VM.
* **NodePool**
    * **Definition:** A cloud-provider concept grouping Nodes of the same size/type.
    * **Why it exists:** Allows a cluster to have mixed hardware (e.g., a pool of GPU nodes for AI, a pool of cheap CPU nodes for web servers).
    * **Prod example:** GCP GKE NodePool with preemptible/spot instances for cheap batch jobs.
    * **CLI:** `gcloud container node-pools list`
    * **Interview Trap:** Looking for NodePools in native K8s docs. It's a managed cloud provider concept, not a native K8s API object.
* **kubelet**
    * **Definition:** The Kubernetes agent running on every Node.
    * **Why it exists:** Takes orders from the API server and talks to the container runtime (like containerd) to start/stop containers.
    * **Prod example:** The kubelet restarting a crashed container locally without bothering the Control Plane.
    * **CLI:** `systemctl status kubelet` (run on the node).
    * **Interview Trap:** Not knowing that kubelet runs as a standard OS daemon (systemd), *not* as a K8s pod (usually).
* **kube-proxy**
    * **Definition:** The network proxy running on every Node.
    * **Why it exists:** Maintains network rules (via iptables or IPVS) allowing network communication to Pods from inside or outside the cluster.
    * **Prod example:** Routing traffic dynamically to a new pod replica when an old one is killed.
    * **CLI:** `kubectl get ds kube-proxy -n kube-system`
    * **Interview Trap:** Believing kube-proxy is a physical load balancer. It’s just a daemon manipulating Linux networking rules.

#### 2. The Core: Execution Units
* **Container**
    * **Definition:** A packaged application with its code and dependencies (e.g., Docker image).
    * **Why it exists:** Guarantees the app runs the same way on a developer's laptop as it does in production.
    * **Prod example:** An NGINX container serving static files.
    * **CLI:** `docker ps` or `crictl ps` (on node).
    * **Interview Trap:** Using "Docker" and "Container" interchangeably. K8s deprecated Docker as a runtime in v1.24+; it now uses CRI-compatible runtimes like containerd.
* **Pod**
    * **Definition:** The smallest deployable unit in K8s; a wrapper around one or more containers.
    * **Why it exists:** Containers in a pod share the same IP address, network port space, and storage volumes.
    * **Prod example:** A Java Spring Boot application container running alongside a Datadog metrics-scraper container in the same pod.
    * **CLI:** `kubectl get pods`
    * **Interview Trap:** Deploying Pods directly. *Trap answer:* "I create pods for my app." *Correction:* You should use a Deployment; bare pods don't self-heal if the node dies.
* **Init Container**
    * **Definition:** A container that runs and *must complete successfully* before the main app container starts.
    * **Why it exists:** Used for setup scripts, database migrations, or waiting for a service to be ready.
    * **Prod example:** An init container that runs `flyway migrate` to update DB schemas before the web app boots.
    * **CLI/YAML:** Found under `initContainers:` in the Pod spec.
    * **Interview Trap:** Not realizing that if an init container fails, the whole Pod gets stuck in `Init:CrashLoopBackOff` and the main app *never* starts.
* **Sidecar Container**
    * **Definition:** A helper container running alongside the main app in the same Pod.
    * **Why it exists:** Offloads tasks like logging, proxying (service mesh), or syncing files without modifying the main app code.
    * **Prod example:** Fluentbit sidecar reading local app logs and shipping them to Elasticsearch.
    * **CLI/YAML:** *Important Update:* As of K8s 1.29 (Official K8s Docs, 2026), native sidecars are defined as `initContainers` with `restartPolicy: Always`.
    * **Interview Trap:** Handling sidecar lifecycle. Before 1.29, K8s didn't natively know which was the "main" container, meaning a sidecar could keep a Job running forever. Native sidecars fix this.

#### 3. Workloads: Controllers
* **ReplicaSet**
    * **Definition:** Ensures a specified number of identical Pod replicas are running.
    * **Why it exists:** Provides high availability. If you ask for 3 and 1 dies, it boots a new one.
    * **Prod example:** Scaling an API from 2 pods to 10 pods during Black Friday.
    * **CLI:** `kubectl get rs`
    * **Interview Trap:** You should almost *never* create a ReplicaSet directly. You create a Deployment, which manages the ReplicaSet for you.
* **Deployment**
    * **Definition:** A declarative update mechanism for ReplicaSets and Pods.
    * **Why it exists:** Allows zero-downtime rolling updates and rollbacks.
    * **Prod example:** Upgrading a web app from v1 to v2 gracefully, swapping out pods one by one.
    * **CLI:** `kubectl create deploy web --image=nginx`
    * **Interview Trap:** Using Deployments for databases. Deployments don't guarantee strict ordering or persistent storage identities (use StatefulSet instead).
* **StatefulSet**
    * **Definition:** Manages stateful applications with stable, persistent identities.
    * **Why it exists:** Databases need stable hostnames (db-0, db-1) and their exact assigned hard drives, even if they restart on a new node.
    * **Prod example:** Running a 3-node MongoDB cluster or Kafka brokers.
    * **CLI:** `kubectl get sts`
    * **Interview Trap:** Forgetting the headless service. A StatefulSet requires a Headless Service to give each pod its predictable DNS name.
* **DaemonSet**
    * **Definition:** Ensures exactly *one* copy of a Pod runs on *every* Node in the cluster.
    * **Why it exists:** For cluster-wide agents like log shippers or network plugins.
    * **Prod example:** Running the Datadog agent or Calico CNI on every server.
    * **CLI:** `kubectl get ds`
    * **Interview Trap:** Wondering how to scale a DaemonSet. You can't change `replicas`. It scales automatically when you add/remove Nodes to the cluster.
* **Job**
    * **Definition:** Runs a pod to completion (does a task, finishes, and terminates).
    * **Why it exists:** For batch processing or one-off scripts.
    * **Prod example:** A nightly script that calculates user billing and shuts down.
    * **CLI:** `kubectl create job my-job --image=busybox`
    * **Interview Trap:** Using a Job for an ongoing server. A Job expects the container to exit with code `0`. If it doesn't, it retries.
* **CronJob**
    * **Definition:** A Job that runs on a time-based schedule (like Linux cron).
    * **Why it exists:** Automates recurring tasks.
    * **Prod example:** Backing up the database to S3 every day at 2 AM.
    * **CLI:** `kubectl get cj`
    * **Interview Trap:** Assuming exact timing. CronJobs run *around* the scheduled time; they can rarely skip a run or run twice depending on `concurrencyPolicy`.

#### 4. Networking: Connecting Things
* **Service**
    * **Definition:** A stable IP address and DNS name that load-balances traffic to a dynamic group of Pods.
    * **Why it exists:** Pod IPs change constantly when they restart. A Service provides a reliable address.
    * **Prod example:** An API pod talking to the `database-service` rather than tracking individual database pod IPs.
    * **CLI:** `kubectl get svc`
    * **Interview Trap:** Services don't find pods by magic; they use `selector` labels. If your pod labels don't match the Service selector, traffic fails silently.
* **ClusterIP**
    * **Definition:** The default Service type. Internal only.
    * **Why it exists:** Secures internal microservices from the public internet.
    * **Prod example:** A backend microservice that only the frontend should talk to.
* **NodePort**
    * **Definition:** Opens a specific static port (30000-32767) on *every* Node's IP.
    * **Why it exists:** Quick and dirty way to expose a service externally, or as a building block for LoadBalancers.
    * **Prod example:** Allowing a developer to hit an API bypassing the load balancer during testing.
    * **Interview Trap:** Highly insecure and hard to manage for real production web traffic.
* **LoadBalancer**
    * **Definition:** Automatically provisions a cloud provider's external load balancer (like AWS ALB/NLB).
    * **Why it exists:** Exposes your app directly to the internet with a public IP.
    * **Prod example:** Your company's main public-facing web server.
    * **Interview Trap:** Cost. Creating 50 LoadBalancer Services provisions 50 expensive cloud load balancers. (Use an Ingress instead).
* **Headless Service**
    * **Definition:** A Service with `clusterIP: None`. It doesn't load balance.
    * **Why it exists:** Instead of returning one IP, DNS returns the IPs of *all* the backing pods. Crucial for StatefulSets so apps can connect to a specific pod (like a database master).
    * **Prod example:** Cassandra nodes discovering each other via DNS.
* **Ingress**
    * **Definition:** A smart HTTP/HTTPS router acting as a single entry point to your cluster.
    * **Why it exists:** Routes traffic based on URL paths (`/api` goes to API pod, `/web` goes to web pod) using only *one* public IP.
    * **Prod example:** `company.com/api` routing to one service and `company.com/blog` routing to another.
    * **CLI:** `kubectl get ingress`
    * **Interview Trap:** Evolving standard. K8s docs show the **Gateway API** is slowly replacing the Ingress API for advanced routing. Mention Gateway API to impress the interviewer.
* **Ingress Controller**
    * **Definition:** The actual software that executes the Ingress rules.
    * **Why it exists:** Ingress is just a rule (a piece of paper); the Controller (like NGINX or Traefik) is the traffic cop that reads the paper and routes the cars.
    * **Prod example:** NGINX Ingress Controller.
* **DNS and CoreDNS**
    * **Definition:** The internal phonebook of the cluster.
    * **Why it exists:** Translates `http://my-database` into an internal IP address.
    * **Prod example:** CoreDNS deployed automatically in the `kube-system` namespace.
    * **Interview Trap:** Not knowing the FQDN structure: `<service-name>.<namespace>.svc.cluster.local`.
* **Service Mesh (Istio, Linkerd)**
    * **Definition:** An infrastructure layer that intercepts all pod-to-pod network traffic using sidecar proxies.
    * **Why it exists:** Provides mutual TLS (mTLS) encryption, retry logic, and deep traffic metrics without changing application code.
    * **Prod example:** Using Istio to encrypt all internal bank transactions between microservices.
    * **Interview Trap:** Over-engineering. Don't use a mesh unless you have massive scale or strict compliance needs; it adds massive complexity and latency.
* **Knative**
    * **Definition:** A platform for Serverless workloads on Kubernetes.
    * **Why it exists:** Scales pods to zero when there is no traffic, and boots them instantly when requests arrive.

#### 5. Storage: Saving Data
* **Volume**
    * **Definition:** A directory containing data, accessible to containers in a pod.
    * **Why it exists:** Because container filesystems are ephemeral. If a container crashes, internal files are gone forever.
* **Ephemeral Volume**
    * **Definition:** Storage tied to the lifecycle of the Pod (e.g., `emptyDir`).
    * **Why it exists:** For cache files or sharing files between an init container and a main container. Dies when the pod dies.
* **PersistentVolume (PV)**
    * **Definition:** A piece of actual storage in the cluster (like an AWS EBS volume or NFS share).
    * **Why it exists:** Exists independently of any Pod.
* **PersistentVolumeClaim (PVC)**
    * **Definition:** A request (claim) by a Pod for storage.
    * **Why it exists:** Abstracts storage. Developers just ask for "10GB of fast storage" via PVC, and K8s finds a matching PV.
    * **Prod example:** A PostgreSQL pod claiming 50GB of SSD storage.
    * **Interview Trap:** Confusing PV with PVC. PV is the *hardware/disk*, PVC is the *ticket* the developer writes to get the disk.
* **StorageClass**
    * **Definition:** A blueprint that provisions PVs dynamically.
    * **Why it exists:** Eliminates the need for admins to manually create PVs. If a PVC asks for "gold" storage, the StorageClass talks to AWS and creates an EBS drive instantly.
* **CSI (Container Storage Interface)**
    * **Definition:** The standard API that allows storage vendors (AWS, NetApp, Ceph) to plug into K8s.
    * **Why it exists:** So K8s core code doesn't have to contain specific logic for every storage array on earth.

#### 6. Configuration & Security
* **Environment Variables & Downward API**
    * **Definition:** Passing config to containers. The Downward API specifically injects K8s metadata (like pod name or namespace) as env vars.
    * **Why it exists:** To make apps dynamic without hardcoding.
* **Namespace**
    * **Definition:** Virtual clusters inside a physical cluster.
    * **Why it exists:** Isolates teams, projects, or environments (e.g., `dev`, `staging`, `prod`).
    * **Interview Trap:** Thinking namespaces isolate security or networking by default. They don't. Pods in `dev` can talk to `prod` unless you add NetworkPolicies.
* **RBAC (Role-Based Access Control)**
    * **Definition:** The K8s permission system.
* **Role & ClusterRole**
    * **Definition:** A list of what actions can be taken (e.g., "can delete pods"). Role is namespace-scoped; ClusterRole is cluster-wide.
* **RoleBinding & ClusterRoleBinding**
    * **Definition:** Connects a Role to a User, Group, or ServiceAccount.
    * **Interview Trap:** Creating a Role, but forgetting to create the RoleBinding. The user will still get `Forbidden`.
* **ServiceAccount**
    * **Definition:** An identity for Pods (machine users), as opposed to human users.
    * **Why it exists:** Lets a Pod securely talk to the K8s API (e.g., a CI/CD pod that needs permission to deploy other pods).
* **Pod Security Standards (PSS)**
    * **Definition:** Built-in policies (Privileged, Baseline, Restricted) to limit what containers can do (e.g., preventing them from running as `root`).
    * **Interview Trap:** Mentioning PodSecurityPolicies (PSP). *Trap answer:* "I use PSPs." *Correction:* PSP was completely removed in v1.25. PSS with Pod Security Admission is the modern standard.
* **NetworkPolicy**
    * **Definition:** A firewall for Pods.
    * **Why it exists:** By default, all K8s pods can talk to each other. NetworkPolicies block traffic (e.g., "Only the API pod can talk to the DB pod").
    * **Interview Trap:** Applying a NetworkPolicy and it doing nothing. Your networking plugin (CNI) *must* support it (e.g., Calico supports them, Flannel does not).
* **Admission Controller & OPA Gatekeeper**
    * **Definition:** Plugins that intercept requests to the API server *before* they are saved.
    * **Why it exists:** To enforce custom company rules.
    * **Prod example:** OPA Gatekeeper blocking any deployment that uses the `:latest` image tag.

#### 7. Scheduling & Scaling
* **Taints and Tolerations**
    * **Definition:** Taints are applied to Nodes ("I am a GPU node, stay away!"). Tolerations are applied to Pods ("I tolerate GPU nodes!").
    * **Why it exists:** Repels pods from specific nodes.
* **Affinity and Anti-affinity**
    * **Definition:** Rules that attract pods to nodes (Node Affinity) or to other pods (Pod Affinity).
    * **Why it exists:** E.g., Anti-affinity ensures 3 replicas of a web server are forced onto 3 *different* physical nodes for High Availability.
* **Resource Requests and Limits**
    * **Definition:** Requests = What a pod *needs* to be scheduled. Limits = The maximum it can *consume* before being throttled or killed.
    * **Interview Trap:** Setting Limits but no Requests, or setting them wildly apart. This causes CPU throttling or OOMKills.
* **QoS Classes**
    * **Definition:** Guaranteed, Burstable, BestEffort.
    * **Why it exists:** Tells K8s which pods to kill first if a node runs out of memory. BestEffort dies first; Guaranteed dies last.
* **HPA (Horizontal Pod Autoscaler)**
    * **Definition:** Scales the *number of pod replicas* based on CPU/RAM or custom metrics.
* **VPA (Vertical Pod Autoscaler)**
    * **Definition:** Scales the *CPU/RAM limits* of a pod by restarting it with bigger sizing.
* **Cluster Autoscaler**
    * **Definition:** Adds or removes physical *Nodes* via the cloud provider when pods are stuck in `Pending` due to lack of space.

#### 8. Observability & Health
* **Liveness, Readiness, Startup Probes**
    * *(See side-by-side comparison below)*
* **Logs & Events**
    * **Logs (`kubectl logs`):** Application stdout/stderr.
    * **Events (`kubectl get events`):** K8s cluster occurrences (e.g., "Pod Scheduled", "Image Pulled").
* **Metrics Server, Prometheus, Grafana**
    * **Metrics Server:** Basic CPU/RAM stats (required for HPA).
    * **Prometheus:** Advanced time-series database scraping deep metrics.
    * **Grafana:** The dashboard UI to visualize Prometheus data.

#### 9. Deployment Strategies & GitOps
* **Helm:** A package manager (like apt/brew) for K8s. Uses Go-templating to parameterize YAML files.
* **Kustomize:** A template-free way to customize K8s YAML by layering patches over a base. Built into `kubectl`.
* **Operators & CRD (Custom Resource Definition):** Teaches K8s new tricks. E.g., creating a `kind: PostgreSQLDatabase` and having a custom Controller (Operator) automatically manage backups.
* **GitOps (Argo CD, Flux):** The practice of storing all YAML in Git. ArgoCD watches Git and immediately forces the K8s cluster to match Git. If someone edits the cluster manually, Argo overwrites it back to what Git says.
* **Deployment Strategies:**
    * **Rolling Update:** One by one replacement (K8s default).
    * **Blue Green:** Stand up an entirely new V2 environment (Green), test it, then flip the router. Instant rollback.
    * **Canary:** Send 5% of user traffic to V2. If no errors, scale to 100%.

---

### C. Side-by-Side Comparisons

| Concept 1 | Concept 2 | Concept 3 | Key Difference |
| :--- | :--- | :--- | :--- |
| **Deployment** | **StatefulSet** | **DaemonSet** | Deployments are for stateless web apps. StatefulSets are for databases needing strict identities/disk. DaemonSets run exactly one pod per physical Node. |
| **ClusterIP** | **NodePort** | **LoadBalancer** | ClusterIP = Internal only. NodePort = Opens port on Node IP. LoadBalancer = Provisions cloud hardware balancer for public access. |
| **ConfigMap** | **Secret** | | ConfigMap stores plain text config. Secret stores base64-encoded sensitive data (passwords). *Note: Secrets are NOT encrypted by default in etcd without extra config.* |
| **HPA** | **VPA** | **Cluster Autoscaler** | HPA scales *Pod count*. VPA scales *Pod size* (CPU/RAM). CA scales *Node count* (VMs). |
| **Liveness** | **Readiness** | **Startup** | Liveness: "Are you dead?" (Restarts pod). Readiness: "Are you busy?" (Removes from load balancer, doesn't restart). Startup: "Are you done booting?" (Pauses others until true). |
| **Pod** | **Container** | | A Container is the engine; a Pod is the car wrapping the engine. K8s manages cars, not engines directly. |

---

### D. Common Errors & How to Debug Them

When a pod fails, use `kubectl describe pod <name>` and look at the **Events** at the bottom.

* **Pending:** The pod is accepted but can't be scheduled.
    * *Cause:* No nodes available, lack of CPU/RAM, or strict nodeSelectors/Taints preventing scheduling.
* **ContainerCreating:** Pod is scheduled, kubelet is setting it up. If stuck:
    * *Cause:* Usually network plugin (CNI) failures or unable to mount a Persistent Volume.
* **ErrImagePull / ImagePullBackOff:**
    * *Cause:* Typo in the image name, image doesn't exist, or K8s lacks permissions/secrets to pull from a private registry.
* **CrashLoopBackOff:**
    * *Cause:* The container starts, but the application immediately crashes (exit code > 0), so K8s restarts it. Over and over.
    * *Fix:* `kubectl logs <pod-name> --previous` to see why the app panicked. Usually a bad config or missing env var.
* **OOMKilled (Out of Memory):**
    * *Cause:* The container exceeded its defined memory `Limit`. The Linux kernel killed it to protect the node.
    * *Fix:* Increase memory limits or fix the app's memory leak.
* **CreateContainerConfigError:**
    * *Cause:* The pod references a ConfigMap or Secret that does not exist.
* **RunContainerError:**
    * *Cause:* The container engine failed to start the container (e.g., trying to mount a volume that fails).
* **Terminating:**
    * *Cause:* Stuck terminating usually means a finalizer is blocking deletion, or a network/disk issue is preventing cleanup. Force with `--force --grace-period=0`.
* **Evicted:**
    * *Cause:* The physical Node ran out of disk space or memory, and aggressively kicked the pod off to save itself.

---

### E. Production Realistic YAML

Here is a production-grade Deployment that connects several concepts:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-api
  namespace: prod
  labels:
    app: payment-api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: payment-api
  template:
    metadata:
      labels:
        app: payment-api
    spec:
      affinity:
        # Anti-affinity: Never put two payment-api pods on the same physical Node
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - payment-api
            topologyKey: "kubernetes.io/hostname"
      containers:
      - name: app
        image: myregistry.com/payment-api:v2.1.0
        ports:
        - containerPort: 8080
        resources:
          requests:
            cpu: "250m"
            memory: "512Mi"
          limits:
            cpu: "500m"
            memory: "1Gi"
        env:
        - name: DB_HOST
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: db_url
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: password
        livenessProbe:
          httpGet:
            path: /healthz
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
```

---

### F. Interview Questions and Model Answers

**Q1: We have an application that requires heavy initialization (running DB migrations) before the web server starts. How would you architect this in Kubernetes?**
* **Model Answer:** I would use an `Init Container`. The init container would contain the DB migration tool and connect to the database. The main application container will not start until the init container completes successfully. If the init container fails, Kubernetes will retry it, keeping the main app safely offline until the schema is ready.

**Q2: You apply a new Deployment YAML, but `kubectl get pods` shows the old pods are still running and no new ones are created. What is likely happening?**
* **Model Answer:** The Deployment is likely failing to roll out due to an invalid image name, a failed readiness probe on the new pods, or insufficient cluster resources causing the new pods to sit in `Pending`. Because of the rolling update strategy, K8s won't kill the old pods until the new ones pass their readiness checks. I would run `kubectl describe deployment` to look at the events and ReplicaSet status.

**Q3: How do you secure pod-to-pod communication in a namespace?**
* **Model Answer:** By default, namespaces are flat networks. To secure them, I would implement `NetworkPolicies` using a CNI that supports them, enforcing a default-deny rule, and explicitly allowing traffic using pod selector labels. If strict encryption in transit is required, I would introduce a Service Mesh like Istio for automated mTLS.

**Q4: Explain the difference between Taints/Tolerations and Node Affinity.**
* **Model Answer:** Taints are defensive; they are placed on Nodes to *repel* pods that don't have matching Tolerations. Node Affinity is offensive; it is placed on Pods to *attract* them to specific nodes. If you want a dedicated GPU node that only allows ML pods, you must use a Taint to keep regular web pods out, and a Toleration on the ML pods to allow them in.

---

### G. Final Cheat Sheet & Memory Tricks

* **The Command Flow:** `kubectl` -> API Server -> etcd -> Controller Manager/Scheduler -> Node's kubelet -> containerd -> Pod.
* **Deployments = Stateless (Web). StatefulSets = Stateful (DBs).**
* **Secret vs ConfigMap:** Secrets are just base64 ConfigMaps. Don't commit either to GitHub without a tool like ExternalSecrets or SealedSecrets.
* **Scaling Trick (H-V-C):**
    * **H**PA = **H**ow many pods?
    * **V**PA = **V**olume/Size of pods?
    * **C**A = **C**omputers (Nodes) in the cluster?
* **Probe Memory Trick:**
    * **L**iveness = **L**ife or death (restarts the pod).
    * **R**eadiness = **R**outing traffic (removes from Service endpoints).
* **ImagePullBackOff?** You made a typo in the image name or lack auth.
* **CrashLoopBackOff?** The container ran, but the code crashed. Check `kubectl logs`.