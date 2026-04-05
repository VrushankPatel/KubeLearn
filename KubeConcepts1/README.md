I'll search for current best practices and interview patterns for Kubernetes, then create a comprehensive guide covering all these concepts.Perfect! Now I'll create a comprehensive, structured guide covering all these concepts. Given the massive list, I'll organize this systematically for interview preparation.

---

# **Complete Kubernetes Interview Preparation Guide**

This is your battle-tested reference covering everything you need for Kubernetes interviews. I'll explain each concept with practical examples, avoiding jargon until it's defined.

---

## **Part 1: Core Workload Concepts**

### **1. Pod (The Foundation)**

**What it is:** The smallest unit you can deploy in Kubernetes. Think of it as a wrapper around one or more containers.

**Real-world analogy:** A pod is like a shared apartment where roommates (containers) live together, sharing utilities (network, storage).

**Key characteristics:**
- Each pod gets its own IP address
- Containers inside share network namespace (can talk via `localhost`)
- Containers inside can share storage volumes
- Pods are ephemeral (temporary, can be killed and recreated)

**Example YAML:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-pod
spec:
  containers:
  - name: nginx
    image: nginx:1.21
    ports:
    - containerPort: 80
```

**Interview tip:** When asked "Why not just use containers?", answer: "Pods allow multiple tightly-coupled containers to share resources. For example, a main app container + a logging sidecar container."

---

### **2. Container vs Init Container vs Sidecar Container**

**Container (Main Container):**
- Your actual application
- Runs for the lifetime of the pod
- Example: Your Perl web app

**Init Container:**
- Runs BEFORE your main containers start
- Runs to completion then exits
- Use case: Database migrations, downloading config files, waiting for dependencies

**Example:**
```yaml
spec:
  initContainers:
  - name: wait-for-db
    image: busybox
    command: ['sh', '-c', 'until nslookup db-service; do sleep 2; done']
  containers:
  - name: app
    image: my-app
```

**What this does:** Pod won't start the main app until the database service is available.

**Sidecar Container:**
- Runs alongside your main container
- Extends or enhances functionality
- Common uses: Log shipping, monitoring, proxies

**Example:**
```yaml
spec:
  containers:
  - name: app
    image: my-app
    volumeMounts:
    - name: logs
      mountPath: /var/log
  - name: log-shipper  # Sidecar
    image: fluentd
    volumeMounts:
    - name: logs
      mountPath: /var/log
  volumes:
  - name: logs
    emptyDir: {}
```

**Interview question:** "When would you use an init container vs a sidecar?"
**Answer:** Init containers for one-time setup tasks before app starts. Sidecars for ongoing auxiliary functions during app lifetime.

---

### **3. ReplicaSet**

**What it is:** Ensures a specified number of pod replicas are running at any time.

**The problem it solves:** You manually create 3 pods. One crashes. Now you have 2. ReplicaSet automatically creates a new one to maintain 3.

**Example:**
```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: web-rs
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: nginx
        image: nginx
```

**How it works:**
1. ReplicaSet controller continuously watches the cluster
2. Counts pods matching the selector (`app: web`)
3. If count < 3, creates more
4. If count > 3, deletes extras

**Interview caveat:** "You almost never create ReplicaSets directly. You use Deployments, which manage ReplicaSets for you."

---

### **4. Deployment (Most Important for Stateless Apps)**

**What it is:** High-level wrapper around ReplicaSets that provides:
- Declarative updates
- Rolling updates
- Rollbacks
- Versioning

**Traditional way:**
```bash
# Stop old app
# Deploy new version
# Hope nothing breaks
# If it breaks, frantically roll back
```

**Kubernetes Deployment way:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-deploy
spec:
  replicas: 5
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: nginx
        image: nginx:1.20
```

**Update process:**
```bash
kubectl set image deployment/web-deploy nginx=nginx:1.21
```

**What happens automatically:**
1. Creates new ReplicaSet with nginx:1.21
2. Starts 1 new pod (maxSurge: 1)
3. Waits for it to be ready
4. Terminates 1 old pod (maxUnavailable: 1)
5. Repeats until all 5 pods are updated
6. **Zero downtime!**

**Rollback:**
```bash
kubectl rollout undo deployment/web-deploy
```

**Interview gold:** "Deployments manage ReplicaSets. When you update a Deployment, it creates a NEW ReplicaSet with the new version and gradually scales it up while scaling down the old ReplicaSet."

---

### **5. StatefulSet (For Stateful Applications)**

**The problem with Deployments:**
- Pods are interchangeable (pod-1234, pod-5678)
- Pods can be killed and recreated with different names
- No guaranteed network identity
- **This breaks databases!**

**What StatefulSet provides:**
- Stable, unique network identifiers (web-0, web-1, web-2)
- Stable persistent storage
- Ordered deployment and scaling
- Ordered updates

**Example: PostgreSQL Cluster**
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres-headless
  replicas: 3
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:13
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 10Gi
```

**Key differences:**
- Pods are named: `postgres-0`, `postgres-1`, `postgres-2`
- `postgres-0` always gets the same persistent volume
- Pods are created in order: 0, then 1, then 2
- Pods are deleted in reverse: 2, then 1, then 0
- Each pod gets its own persistent volume

**Interview question:** "When to use StatefulSet vs Deployment?"
**Answer:** 
- **StatefulSet:** Databases, message queues, anything needing stable identity/storage
- **Deployment:** Stateless apps like web servers, APIs

---

### **6. DaemonSet**

**What it is:** Ensures a copy of a pod runs on EVERY node (or selected nodes).

**Use cases:**
- Log collectors (need logs from every machine)
- Monitoring agents (need metrics from every machine)
- Storage daemons
- Network plugins

**Example: Node monitoring agent**
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-monitor
spec:
  selector:
    matchLabels:
      app: monitor
  template:
    metadata:
      labels:
        app: monitor
    spec:
      containers:
      - name: prometheus-node-exporter
        image: prom/node-exporter
```

**Behavior:**
- New node added to cluster? DaemonSet automatically creates a pod there
- Node removed? Pod is cleaned up
- You have 10 nodes? You get exactly 10 pods

**Interview insight:** "DaemonSets ignore unschedulable nodes and respect node taints/tolerations for targeting specific nodes."

---

### **7. Job**

**What it is:** Runs a pod to completion, then stops.

**Use case:** Batch processing, one-time tasks, database migrations

**Example: Database backup**
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-backup
spec:
  template:
    spec:
      containers:
      - name: backup
        image: backup-tool
        command: ['./backup.sh']
      restartPolicy: OnFailure  # Important!
  backoffLimit: 4  # Retry 4 times if it fails
```

**Key points:**
- Pod runs once
- If it fails, Job retries (up to `backoffLimit`)
- When successful, pod stays in "Completed" state (for logs)
- You can run parallel jobs: `parallelism: 3` (runs 3 pods simultaneously)

---

### **8. CronJob**

**What it is:** Scheduled Jobs (like cron in Linux).

**Example: Daily database backup**
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: daily-backup
spec:
  schedule: "0 2 * * *"  # 2 AM daily
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup
            image: backup-tool
            command: ['./backup.sh']
          restartPolicy: OnFailure
```

**Interview tip:** Schedule format is standard cron: `minute hour day month weekday`

---

## **Part 2: Networking Concepts**

### **9. Service (Stable Network Access)**

**The problem:** 
- Pods have IPs, but they're ephemeral
- Pod dies and gets recreated → new IP
- You have 5 pods → which IP to use?

**Service solution:** Provides a stable endpoint

**Four Types:**

#### **A. ClusterIP (Default, Internal Only)**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-service
spec:
  type: ClusterIP
  selector:
    app: backend
  ports:
  - port: 80
    targetPort: 8080
```

**What it does:**
- Gets a stable internal IP (e.g., `10.96.0.10`)
- Only accessible inside the cluster
- Automatically load-balances to all pods with label `app: backend`

#### **B. NodePort (External Access via Node IP)**
```yaml
spec:
  type: NodePort
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 8080
    nodePort: 30080  # Optional, range: 30000-32767
```

**What it does:**
- Opens port 30080 on EVERY node
- You can access it via `<any-node-ip>:30080`
- Traffic gets load-balanced to pods

**Use case:** Testing, simple external access

#### **C. LoadBalancer (Cloud Load Balancer)**
```yaml
spec:
  type: LoadBalancer
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 8080
```

**What it does:**
- On AWS: Creates an ELB/ALB
- On GCP: Creates a GCP Load Balancer
- On Azure: Creates an Azure Load Balancer
- You get a public IP/DNS

**Use case:** Production web apps needing external access

#### **D. Headless Service (No Load Balancing)**
```yaml
spec:
  clusterIP: None  # This makes it headless
  selector:
    app: database
  ports:
  - port: 5432
```

**What it does:**
- No single cluster IP
- Returns IPs of ALL matching pods
- Used with StatefulSets for direct pod access

**Example:** Database clients can connect to specific pods: `postgres-0.postgres-headless`, `postgres-1.postgres-headless`

---

### **10. Ingress (Smart HTTP/HTTPS Routing)**

**The problem:**
- You have 5 services (web, api, admin, docs, auth)
- LoadBalancer service costs money (1 per service = 5 load balancers!)
- Need path-based routing: `/api` → api-service, `/admin` → admin-service

**Ingress solution:** Single entry point with routing rules

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: main-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: myapp.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
      - path: /admin
        pathType: Prefix
        backend:
          service:
            name: admin-service
            port:
              number: 80
  - host: blog.myapp.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: blog-service
            port:
              number: 80
```

**What this does:**
- `myapp.com/api/*` → routes to api-service
- `myapp.com/admin/*` → routes to admin-service
- `blog.myapp.com/*` → routes to blog-service

**TLS/HTTPS:**
```yaml
spec:
  tls:
  - hosts:
    - myapp.com
    secretName: tls-cert-secret
  rules:
  ...
```

---

### **11. Ingress Controller**

**Critical understanding:** Ingress resource by itself does NOTHING!

You need an **Ingress Controller** (actual implementation):
- **NGINX Ingress Controller** (most popular)
- **Traefik**
- **HAProxy**
- **AWS ALB Ingress Controller**

**Installation (example):**
```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.0/deploy/static/provider/cloud/deploy.yaml
```

**Interview question:** "What's the difference between Ingress and Ingress Controller?"
**Answer:** "Ingress is the configuration (routes). Ingress Controller is the actual software (like nginx) that implements those routes."

---

### **12. DNS (CoreDNS)**

**Automatic DNS in Kubernetes:**
- Every Service gets a DNS name
- Format: `<service-name>.<namespace>.svc.cluster.local`

**Example:**
```yaml
# Service in "production" namespace
apiVersion: v1
kind: Service
metadata:
  name: database
  namespace: production
```

**DNS names:**
- Full: `database.production.svc.cluster.local`
- Short (from same namespace): `database`
- From other namespace: `database.production`

**Pods also get DNS:**
- Format: `<pod-ip-with-dashes>.<namespace>.pod.cluster.local`
- Example: `10-244-1-5.default.pod.cluster.local`

**Interview insight:** "Services use CoreDNS for service discovery. You don't need to hardcode IPs; just use service names."

---

### **13. kube-proxy (Network Rules)**

**What it does:** Implements Service networking on each node using iptables or IPVS.

**How it works:**
1. Service created with ClusterIP `10.96.0.10`
2. kube-proxy on each node creates iptables rules
3. Traffic to `10.96.0.10:80` gets redirected to actual pod IPs
4. Load balancing happens at the network level

**You don't interact with kube-proxy directly**, but it's crucial for Services to work.

---

### **14. Service Mesh (Istio, Linkerd)**

**The problem Services don't solve:**
- Encrypted traffic between services
- Advanced routing (canary deployments, A/B testing)
- Distributed tracing
- Fine-grained traffic control

**Service Mesh solution:** Adds a sidecar proxy to every pod

**Example: Istio**
- Injects an Envoy proxy sidecar into each pod
- All traffic goes through the proxy
- Provides: mTLS, retries, circuit breaking, metrics, tracing

**Interview scope:** Know it exists, basic concept. Deep knowledge only needed for senior/specialized roles.

---

### **15. NetworkPolicy (Firewall Rules)**

**Default behavior:** All pods can talk to all pods.

**Problem:** Your database should only accept connections from your API, not from random pods.

**NetworkPolicy example:**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-policy
spec:
  podSelector:
    matchLabels:
      app: database
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: api
    ports:
    - protocol: TCP
      port: 5432
```

**What this does:** Only pods with label `app: api` can connect to pods with label `app: database` on port 5432.

---

## **Part 3: Storage**

### **16. Volume (Temporary Storage)**

**Types:**

**emptyDir** (Temporary, pod-specific):
```yaml
volumes:
- name: cache
  emptyDir: {}
```
- Created when pod starts
- Deleted when pod dies
- Shared among containers in the pod

**hostPath** (Node's filesystem):
```yaml
volumes:
- name: logs
  hostPath:
    path: /var/log
```
- Mounts directory from the node
- **Dangerous:** Pod tied to specific node
- Use case: Accessing Docker socket, node logs

---

### **17. PersistentVolume (PV) & PersistentVolumeClaim (PVC)**

**Concept separation:**
- **PV:** Actual storage (admin creates this)
- **PVC:** Request for storage (user creates this)

**Admin creates PV:**
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-nfs
spec:
  capacity:
    storage: 10Gi
  accessModes:
  - ReadWriteMany
  nfs:
    server: nfs-server.example.com
    path: /exports
```

**User creates PVC:**
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-storage
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 5Gi
```

**Use in Pod:**
```yaml
volumes:
- name: data
  persistentVolumeClaim:
    claimName: my-storage
```

**Access Modes:**
- `ReadWriteOnce` (RWO): One node can mount read-write
- `ReadOnlyMany` (ROX): Many nodes can mount read-only
- `ReadWriteMany` (RWX): Many nodes can mount read-write (NFS, CephFS)

---

### **18. StorageClass (Dynamic Provisioning)**

**Old way:** Admin manually creates PVs. User creates PVC. Kubernetes binds them.

**New way (Dynamic):** User creates PVC → Kubernetes automatically provisions storage

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp3
  iops: "3000"
```

**User's PVC:**
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: db-storage
spec:
  storageClassName: fast-ssd
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
```

**What happens:**
1. PVC created
2. StorageClass sees it
3. Calls AWS API to create a 20GB gp3 EBS volume
4. Creates PV automatically
5. Binds PVC to PV
6. Pod can use it

---

### **19. CSI (Container Storage Interface)**

**What it is:** Standard plugin interface for storage systems.

**Why it matters:** Cloud providers and storage vendors can create CSI drivers to integrate with Kubernetes.

**Examples:**
- AWS EBS CSI Driver
- Google Persistent Disk CSI Driver
- Ceph CSI
- NFS CSI

**You don't write CSI drivers**, but knowing they exist shows understanding of how Kubernetes integrates with storage.

---

### **20. Ephemeral Volume**

**New in Kubernetes:** Volumes that exist only for the pod's lifetime but aren't `emptyDir`.

**Example: Generic ephemeral volume**
```yaml
volumes:
- name: scratch
  ephemeral:
    volumeClaimTemplate:
      spec:
        accessModes: [ "ReadWriteOnce" ]
        resources:
          requests:
            storage: 1Gi
```

**Use case:** Temporary storage that needs specific StorageClass features.

---

## **Part 4: Configuration**

### **21. ConfigMap (Non-Sensitive Config)**

**Use case:** Store configuration that your app needs

**Create ConfigMap:**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  database_url: "postgres://db.example.com:5432/mydb"
  max_connections: "100"
  log_level: "info"
  config.json: |
    {
      "feature_flags": {
        "new_ui": true
      }
    }
```

**Use in Pod (Environment Variables):**
```yaml
spec:
  containers:
  - name: app
    image: my-app
    env:
    - name: DB_URL
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: database_url
    - name: MAX_CONN
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: max_connections
```

**Use as Volume (Files):**
```yaml
spec:
  containers:
  - name: app
    image: my-app
    volumeMounts:
    - name: config
      mountPath: /etc/config
  volumes:
  - name: config
    configMap:
      name: app-config
```

Result: `/etc/config/database_url`, `/etc/config/config.json`, etc.

---

### **22. Secret (Sensitive Data)**

**Like ConfigMap but for sensitive data**: passwords, API keys, TLS certs

**Create Secret:**
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
data:
  password: cGFzc3dvcmQxMjM=  # base64 encoded
  api-key: YXBpa2V5eHl6  # base64 encoded
```

**Create from command:**
```bash
kubectl create secret generic db-secret \
  --from-literal=password=password123 \
  --from-literal=api-key=apikeyxyz
```

**Use in Pod:**
```yaml
spec:
  containers:
  - name: app
    image: my-app
    env:
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: password
```

**TLS Secret (for Ingress):**
```bash
kubectl create secret tls tls-cert \
  --cert=path/to/tls.crt \
  --key=path/to/tls.key
```

**Interview warning:** "Secrets are base64 encoded, NOT encrypted by default. Enable encryption at rest for production."

---

### **23. Environment Variables**

Three ways to set them:

**1. Direct:**
```yaml
env:
- name: LOG_LEVEL
  value: "debug"
```

**2. From ConfigMap:**
```yaml
env:
- name: DB_URL
  valueFrom:
    configMapKeyRef:
      name: app-config
      key: database_url
```

**3. From Secret:**
```yaml
env:
- name: API_KEY
  valueFrom:
    secretKeyRef:
      name: api-secret
      key: key
```

---

### **24. Downward API (Pod/Container Info)**

**Access pod metadata as environment variables or files:**

**Environment variables:**
```yaml
env:
- name: MY_POD_NAME
  valueFrom:
    fieldRef:
      fieldPath: metadata.name
- name: MY_POD_NAMESPACE
  valueFrom:
    fieldRef:
      fieldPath: metadata.namespace
- name: MY_POD_IP
  valueFrom:
    fieldRef:
      fieldPath: status.podIP
- name: MY_CPU_LIMIT
  valueFrom:
    resourceFieldRef:
      containerName: app
      resource: limits.cpu
```

**Use case:** App needs to know its own pod name for logging.

---

## **Part 5: Scheduling & Resource Management**

### **25. Scheduler**

**What it does:** Decides which node a pod should run on.

**Factors considered:**
- Resource requests (CPU, memory)
- Node selectors
- Affinity/anti-affinity
- Taints and tolerations
- Pod priority
- Topology spread

**You don't usually interact with the scheduler directly**, but understanding its role is crucial.

---

### **26. Node & NodePool**

**Node:** A worker machine (VM or physical) in the cluster.

**NodePool (Cloud-specific):** Group of nodes with the same configuration.

**Example in GKE:**
```bash
gcloud container node-pools create high-mem-pool \
  --cluster=my-cluster \
  --machine-type=n1-highmem-4 \
  --num-nodes=3
```

**Use case:** Different workloads need different machine types (CPU-optimized, memory-optimized, GPU).

---

### **27. Resource Requests & Limits**

**Requests:** Minimum guaranteed resources
**Limits:** Maximum allowed resources

```yaml
resources:
  requests:
    memory: "256Mi"
    cpu: "500m"  # 0.5 CPU cores
  limits:
    memory: "512Mi"
    cpu: "1000m"  # 1 CPU core
```

**What happens:**
- **Scheduler** uses requests to decide if a node can fit the pod
- **kubelet** enforces limits (kills pod if it exceeds memory limit)

**Interview critical:** "CPU is throttled when limit is hit. Memory causes pod to be killed (OOMKilled)."

---

### **28. QoS Classes (Quality of Service)**

**Kubernetes assigns QoS based on requests/limits:**

**Guaranteed (Highest priority):**
```yaml
resources:
  requests:
    memory: "256Mi"
    cpu: "500m"
  limits:
    memory: "256Mi"  # Same as requests
    cpu: "500m"      # Same as requests
```

**Burstable (Medium priority):**
```yaml
resources:
  requests:
    memory: "128Mi"
  limits:
    memory: "512Mi"  # Different from requests
```

**BestEffort (Lowest priority):**
```yaml
# No requests or limits specified
```

**Why it matters:** When node runs out of memory, BestEffort pods are killed first, then Burstable, then Guaranteed.

---

### **29. Taints & Tolerations**

**Problem:** You have GPU nodes. You don't want regular pods wasting them.

**Solution:**
1. **Taint** the GPU nodes (repels pods)
2. Only pods with matching **tolerations** can schedule there

**Taint a node:**
```bash
kubectl taint nodes gpu-node gpu=true:NoSchedule
```

**Pod with toleration:**
```yaml
spec:
  tolerations:
  - key: "gpu"
    operator: "Equal"
    value: "true"
    effect: "NoSchedule"
  containers:
  - name: ml-app
    image: tensorflow
```

**Taint effects:**
- `NoSchedule`: Don't schedule new pods here
- `PreferNoSchedule`: Avoid scheduling here if possible
- `NoExecute`: Don't schedule AND evict existing pods

**Interview question:** "Taints are on nodes. Tolerations are on pods. It's like a lock and key."

---

### **30. Affinity / Anti-affinity**

**More flexible than taints/tolerations.**

**Node Affinity (Schedule based on node labels):**
```yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: disktype
          operator: In
          values:
          - ssd
```

**Result:** Pod MUST run on nodes with label `disktype=ssd`.

**Pod Affinity (Schedule near other pods):**
```yaml
affinity:
  podAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
    - labelSelector:
        matchLabels:
          app: cache
      topologyKey: kubernetes.io/hostname
```

**Result:** This pod MUST run on the same node as pods with label `app=cache`.

**Pod Anti-affinity (Spread pods apart):**
```yaml
affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
    - labelSelector:
        matchLabels:
          app: web
      topologyKey: kubernetes.io/hostname
```

**Result:** Don't put two `app=web` pods on the same node (for high availability).

---

### **31. Horizontal Pod Autoscaler (HPA)**

**Automatically scales pods based on metrics.**

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-deploy
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

**What it does:**
- Monitors CPU usage of pods in `web-deploy`
- If average CPU > 70%, adds more pods
- If average CPU < 70%, removes pods
- Always maintains between 2 and 10 pods

**Requires:** Metrics Server installed in the cluster.

---

### **32. Vertical Pod Autoscaler (VPA)**

**Adjusts resource requests/limits automatically.**

**Scenario:** You set `requests: cpu: 100m`, but your app actually needs 500m.

**VPA solution:** Monitors actual usage and updates requests/limits.

**Interview note:** Less commonly used than HPA. Know it exists.

---

### **33. Cluster Autoscaler**

**Adds/removes nodes based on pending pods.**

**Scenario:**
- You have 3 nodes, all full
- HPA wants to create more pods
- Pods are "Pending" (can't be scheduled)
- Cluster Autoscaler adds a new node
- Pods get scheduled

**Cloud-specific** (works with AWS Auto Scaling Groups, GCP Managed Instance Groups, etc.)

---

## **Part 6: Security & Access Control**

### **34. Namespace (Isolation)**

**Logical partitioning of the cluster.**

```bash
kubectl create namespace production
kubectl create namespace development
```

**Use in resources:**
```yaml
metadata:
  name: web-app
  namespace: production
```

**Benefits:**
- Resource isolation
- RBAC per namespace
- Resource quotas per namespace
- Organizational structure

**Default namespaces:**
- `default`: Where resources go if no namespace specified
- `kube-system`: Kubernetes system components
- `kube-public`: Public resources
- `kube-node-lease`: Node heartbeats

---

### **35. RBAC (Role-Based Access Control)**

**Four concepts:**
1. **Role:** Permissions in a namespace
2. **ClusterRole:** Permissions cluster-wide
3. **RoleBinding:** Binds Role to users/groups in a namespace
4. **ClusterRoleBinding:** Binds ClusterRole to users/groups cluster-wide

**Example: Developer can view pods in "dev" namespace**

**Role:**
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: dev
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
```

**RoleBinding:**
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: dev
subjects:
- kind: User
  name: jane
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

**ClusterRole (cluster-wide):**
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-reader
rules:
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get", "list"]
```

**Verbs:** `get`, `list`, `watch`, `create`, `update`, `patch`, `delete`

---

### **36. ServiceAccount**

**Identity for pods** (not humans).

**Every pod runs with a ServiceAccount.**

**Example:** App needs to query Kubernetes API

**Create ServiceAccount:**
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-sa
```

**Create Role:**
```yaml
kind: Role
metadata:
  name: pod-lister
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["list"]
```

**Create RoleBinding:**
```yaml
kind: RoleBinding
metadata:
  name: app-pod-lister
subjects:
- kind: ServiceAccount
  name: app-sa
roleRef:
  kind: Role
  name: pod-lister
```

**Use in Pod:**
```yaml
spec:
  serviceAccountName: app-sa
  containers:
  - name: app
    image: my-app
```

Now the app can list pods using the Kubernetes API from inside the pod!

---

### **37. Pod Security Standards**

**Three levels:**
1. **Privileged:** Unrestricted (can do anything)
2. **Baseline:** Minimal restrictions (prevents known privilege escalations)
3. **Restricted:** Heavily restricted (best practices)

**Applied at namespace level:**
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted
```

**Interview:** "Replaced Pod Security Policies (deprecated). Know the three levels."

---

### **38. Admission Controllers**

**Intercept API requests before they're persisted.**

**Built-in examples:**
- `NamespaceLifecycle`: Prevents deletion of system namespaces
- `ResourceQuota`: Enforces quotas
- `PodSecurityPolicy`: (Deprecated, replaced by Pod Security Standards)
- `MutatingAdmissionWebhook`: Can modify requests
- `ValidatingAdmissionWebhook`: Can reject requests

**Example use:** Custom admission webhook that rejects pods without resource limits.

---

### **39. OPA Gatekeeper**

**Policy enforcement using Open Policy Agent.**

**Example policy:** All pods must have resource limits.

```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredResources
metadata:
  name: must-have-limits
spec:
  match:
    kinds:
    - apiGroups: [""]
      kinds: ["Pod"]
```

**Interview scope:** Know it exists for advanced policy enforcement.

---

## **Part 7: Observability & Debugging**

### **40. Logs**

**View container logs:**
```bash
kubectl logs pod-name
kubectl logs pod-name -c container-name  # Multi-container pod
kubectl logs -f pod-name  # Follow (tail -f)
kubectl logs --previous pod-name  # Previous crashed container
```

**Interview tip:** "Logs are ephemeral. Use log aggregation (ELK, Loki) for production."

---

### **41. Events**

**Cluster events (pod scheduling, errors, etc.):**
```bash
kubectl get events
kubectl get events --sort-by='.lastTimestamp'
kubectl describe pod pod-name  # Shows events for this pod
```

**Events are crucial for debugging!**

---

### **42. Metrics Server**

**Collects resource metrics (CPU, memory).**

**Install:**
```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

**View metrics:**
```bash
kubectl top nodes
kubectl top pods
```

**Required for:** HPA, VPA, `kubectl top` command

---

### **43. Prometheus & Grafana**

**Prometheus:** Metrics collection and alerting
**Grafana:** Visualization dashboards

**Standard monitoring stack in Kubernetes.**

**Interview scope:** Know they're the standard tools. Basic understanding of their role.

---

### **44. Probes (Health Checks)**

**Three types:**

**A. Liveness Probe (Is the container alive?)**
```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10
```

**If fails:** Container is restarted

**B. Readiness Probe (Is the container ready for traffic?)**
```yaml
readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
```

**If fails:** Pod removed from Service endpoints (no traffic sent to it)

**C. Startup Probe (For slow-starting containers)**
```yaml
startupProbe:
  httpGet:
    path: /started
    port: 8080
  failureThreshold: 30
  periodSeconds: 10
```

**If fails:** Container is killed

**Probe types:**
- `httpGet`: HTTP GET request
- `tcpSocket`: TCP connection
- `exec`: Run a command

**Interview gold:** "Liveness restarts container. Readiness removes from load balancing. Startup gives slow apps time to start before liveness kicks in."

---

### **45. kubectl Commands for Debugging**

**describe (Detailed info + events):**
```bash
kubectl describe pod pod-name
kubectl describe node node-name
```

**logs:**
```bash
kubectl logs pod-name
kubectl logs -f pod-name  # Follow
kubectl logs pod-name --previous  # Previous instance
```

**exec (Get shell inside container):**
```bash
kubectl exec -it pod-name -- /bin/bash
kubectl exec -it pod-name -c container-name -- /bin/sh
```

**port-forward (Access pod locally):**
```bash
kubectl port-forward pod-name 8080:80
# Access via localhost:8080
```

**get (List resources):**
```bash
kubectl get pods
kubectl get pods -o wide  # More info
kubectl get pods -o yaml  # Full YAML
kubectl get all  # All resources
```

**delete:**
```bash
kubectl delete pod pod-name
kubectl delete -f deployment.yaml
```

**apply (Create/update):**
```bash
kubectl apply -f manifest.yaml
```

---

## **Part 8: Common Error States**

### **46. ErrImagePull / ImagePullBackOff**

**Meaning:** Can't download the container image

**Causes:**
1. Image doesn't exist
2. Wrong image name/tag
3. Private registry (no credentials)
4. Network issue

**Debug:**
```bash
kubectl describe pod pod-name
# Look for: "Failed to pull image"
```

**Fix:**
```yaml
# For private registry
imagePullSecrets:
- name: registry-secret
```

---

### **47. CrashLoopBackOff**

**Meaning:** Container starts, crashes, restart, crashes again...

**Causes:**
1. App crashes on startup
2. Misconfiguration
3. Missing dependencies

**Debug:**
```bash
kubectl logs pod-name
kubectl logs pod-name --previous  # See why it crashed
kubectl describe pod pod-name
```

**Interview tip:** "Check logs immediately. Often app errors or missing environment variables."

---

### **48. OOMKilled (Out Of Memory)**

**Meaning:** Container exceeded memory limit and was killed

**Debug:**
```bash
kubectl describe pod pod-name
# Look for: "Reason: OOMKilled"
```

**Fix:** Increase memory limit or fix memory leak in app

---

### **49. CreateContainerConfigError**

**Meaning:** Problem with ConfigMap or Secret

**Causes:**
1. Referenced ConfigMap doesn't exist
2. Referenced Secret doesn't exist
3. Wrong key name

**Debug:**
```bash
kubectl describe pod pod-name
# Look for: "Error: configmap "xyz" not found"
```

---

### **50. Pending**

**Meaning:** Pod can't be scheduled

**Causes:**
1. Not enough resources on any node
2. Node selector doesn't match any node
3. Taints without tolerations
4. PVC not bound

**Debug:**
```bash
kubectl describe pod pod-name
# Look at Events section
```

---

### **51. ContainerCreating**

**Normal status while starting.** If stuck:

**Causes:**
1. Image pull in progress
2. Volume mounting issues
3. CNI network plugin issues

---

### **52. Terminating**

**Pod is shutting down.** If stuck:

**Causes:**
1. Graceful shutdown taking time (respects `terminationGracePeriodSeconds`)
2. Finalizers blocking deletion

**Force delete:**
```bash
kubectl delete pod pod-name --grace-period=0 --force
```

---

### **53. Evicted**

**Meaning:** Node ran out of resources; pod was evicted

**Causes:**
1. Node disk pressure
2. Node memory pressure
3. Node PID pressure

**Check:**
```bash
kubectl get pods | grep Evicted
kubectl describe node node-name
```

---

## **Part 9: Deployment Strategies**

### **54. Rolling Update (Default)**

**Gradually replaces old pods with new ones.**

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 1
    maxSurge: 1
```

**Process:**
1. Start 1 new pod
2. Wait for it to be ready
3. Terminate 1 old pod
4. Repeat

**Zero downtime ✓**

---

### **55. Blue-Green Deployment**

**Two complete environments: Blue (current) and Green (new)**

**Process:**
1. Deploy new version (Green)
2. Test it
3. Switch Service selector from Blue to Green
4. Instant switch
5. Keep Blue for quick rollback

```yaml
# Blue deployment
labels:
  app: web
  version: blue

# Green deployment
labels:
  app: web
  version: green

# Service points to blue initially
selector:
  app: web
  version: blue

# Switch to green
selector:
  app: web
  version: green
```

**Pros:** Instant rollback, safe testing
**Cons:** Double resources during deployment

---

### **56. Canary Deployment**

**Route small % of traffic to new version.**

**Process:**
1. Deploy new version with 1 pod
2. Old version has 9 pods
3. Service load-balances: 10% new, 90% old
4. Monitor metrics
5. If good, scale up new, scale down old

```yaml
# Old version: 9 replicas
# New version: 1 replica
# Both match Service selector
# Natural load balancing gives 10% to new
```

**Better:** Use Istio/Linkerd for precise traffic splitting

---

## **Part 10: Advanced Tools**

### **57. Helm (Package Manager)**

**Like apt/yum for Kubernetes.**

**Chart:** Package of Kubernetes manifests

**Example:**
```bash
# Install nginx
helm install my-nginx bitnami/nginx

# Install with custom values
helm install my-db bitnami/postgresql \
  --set postgresqlPassword=secretpassword

# Upgrade
helm upgrade my-nginx bitnami/nginx

# Rollback
helm rollback my-nginx 1
```

**Custom Chart structure:**
```
mychart/
  Chart.yaml      # Metadata
  values.yaml     # Default values
  templates/      # K8s manifests with templating
    deployment.yaml
    service.yaml
```

**Interview:** "Helm simplifies complex deployments and allows parameterization via values.yaml."

---

### **58. Kustomize (Configuration Management)**

**Built into kubectl. Template-free customization.**

```
base/
  deployment.yaml
  service.yaml
overlays/
  production/
    kustomization.yaml
    patch.yaml
  staging/
    kustomization.yaml
    patch.yaml
```

**Apply:**
```bash
kubectl apply -k overlays/production/
```

**Interview:** "Kustomize is built into kubectl. Helm has more features but adds complexity."

---

### **59. Operators & CRD**

**Operator:** Application-specific controller that extends Kubernetes

**Example:** PostgreSQL Operator

**CRD (Custom Resource Definition):**
```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: postgresqls.database.example.com
spec:
  group: database.example.com
  names:
    kind: PostgreSQL
    plural: postgresqls
  scope: Namespaced
```

**Now you can create:**
```yaml
apiVersion: database.example.com/v1
kind: PostgreSQL
metadata:
  name: my-db
spec:
  version: "13"
  replicas: 3
  storageSize: 20Gi
```

**Operator watches for PostgreSQL resources and creates StatefulSets, Services, etc. automatically.**

---

### **60. GitOps (ArgoCD, Flux)**

**Principle:** Git is the source of truth. Cluster state matches Git repo.

**ArgoCD:**
1. Point ArgoCD at your Git repo
2. ArgoCD continuously monitors
3. Detects changes in Git
4. Automatically applies to cluster
5. Shows drift if cluster differs from Git

**Benefits:**
- Declarative
- Audit trail (Git history)
- Easy rollbacks (Git revert)
- Multi-cluster management

---

## **Part 11: Control Plane Deep Dive**

### **61. API Server (kube-apiserver)**

**Central hub for EVERYTHING.**

**Responsibilities:**
- Exposes Kubernetes API (REST)
- Authentication & authorization
- Admission control
- Stores data in etcd
- Watch mechanism (stream changes)

**All components talk to the API server:**
- kubectl → API Server
- Scheduler → API Server
- Controller Manager → API Server
- kubelet → API Server

---

### **62. etcd**

**Distributed key-value store. The database of Kubernetes.**

**What's stored:**
- All cluster configuration
- All resource definitions (Pods, Services, etc.)
- Secrets
- Current state

**Critical:** Backup etcd regularly!

```bash
# Backup
ETCDCTL_API=3 etcdctl snapshot save backup.db

# Restore
ETCDCTL_API=3 etcdctl snapshot restore backup.db
```

---

### **63. Controller Manager**

**Runs multiple controllers:**

- **Node Controller:** Monitors node health
- **Replication Controller:** Maintains correct pod count
- **Endpoints Controller:** Populates Endpoints (Service → Pods mapping)
- **Service Account Controller:** Creates default ServiceAccounts
- **Namespace Controller:** Handles namespace lifecycle

**Pattern:** Watch → Compare → Act (Reconciliation loop)

---

### **64. Scheduler (kube-scheduler)**

**Watches for unscheduled pods and assigns them to nodes.**

**Scoring process:**
1. **Filtering:** Eliminate nodes that don't meet requirements
2. **Scoring:** Rank remaining nodes
3. **Binding:** Assign pod to highest-scoring node

**Factors:**
- Resource availability
- Node selector
- Affinity/anti-affinity
- Taints/tolerations

---

### **65. kubelet**

**Agent on each worker node.**

**Responsibilities:**
- Watches API Server for pods assigned to its node
- Starts/stops containers (via container runtime)
- Reports pod status back to API Server
- Runs liveness/readiness probes
- Mounts volumes

---

## **Part 12: CI/CD & Immutable Infrastructure**

### **66. Immutable Infrastructure**

**Principle:** Never modify running systems. Always deploy new versions.

**Traditional:**
```bash
ssh server
apt-get update
systemctl restart app
```

**Immutable:**
```bash
# Build new container image: v2.0
# Deploy new pods with v2.0
# Old pods are deleted
```

**Benefits:**
- Predictable deployments
- Easy rollbacks
- No configuration drift

---

### **67. Infrastructure as Code**

**Principle:** Define infrastructure in code (version controlled).

**Kubernetes manifests are IaC:**
```yaml
# This file defines your infrastructure
apiVersion: apps/v1
kind: Deployment
...
```

**Tools:**
- Terraform (for cloud resources + Kubernetes)
- Pulumi
- Helm
- Kustomize

---

### **68. CI/CD Pipeline (Example)**

**Modern pipeline:**

```
1. Developer pushes code to Git
   ↓
2. CI builds Docker image
   ↓
3. CI tags image: my-app:commit-sha
   ↓
4. CI pushes image to registry
   ↓
5. CI updates Kubernetes manifest (or Helm values) in Git
   ↓
6. ArgoCD detects Git change
   ↓
7. ArgoCD deploys to Kubernetes
   ↓
8. Rolling update happens
```

**Tools:**
- CI: Jenkins, GitLab CI, GitHub Actions, CircleCI
- CD: ArgoCD, Flux, Spinnaker
- Registry: Docker Hub, ECR, GCR, Harbor

---

## **Interview Preparation Tips**

### **Study Strategy:**

1. **Hands-on practice** (Most critical!)
   - Install Minikube or Kind
   - Deploy real apps
   - Break things intentionally
   - Fix them

2. **Common scenarios:**
   - Deploy a multi-tier app (web + API + database)
   - Set up Ingress with TLS
   - Configure autoscaling
   - Implement RBAC
   - Debug failing pods

3. **Know your "why":**
   - Don't just memorize commands
   - Understand WHY each component exists
   - What problem does it solve?

4. **Practice kubectl:**
   ```bash
   # Imperative (quick)
   kubectl run nginx --image=nginx
   kubectl create deployment web --image=nginx --replicas=3
   kubectl expose deployment web --port=80 --type=LoadBalancer
   
   # Declarative (production)
   kubectl apply -f deployment.yaml
   kubectl delete -f deployment.yaml
   ```

5. **Common interview questions:**
   - "Explain Kubernetes architecture"
   - "How do you troubleshoot a pod in CrashLoopBackOff?"
   - "Difference between Deployment and StatefulSet?"
   - "How does Service discovery work?"
   - "Explain the pod lifecycle"
   - "How would you secure a cluster?"
   - "How do you do zero-downtime deployments?"

### **Red Flags to Avoid:**
- ❌ "I always use Helm charts from the internet without understanding them"
- ❌ "I don't use resource limits"
- ❌ "I run everything in the default namespace"
- ❌ "I don't know what's in my container images"

### **Green Flags:**
- ✅ "I use resource requests/limits for predictable scheduling"
- ✅ "I implement proper health checks"
- ✅ "I use namespaces for isolation"
- ✅ "I version my manifests in Git"
- ✅ "I monitor metrics and logs"
