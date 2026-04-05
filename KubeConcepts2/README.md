Below is a current, doc based map as of 5 Apr 2026. The core mental model is simple. Kubernetes keeps desired state in the API server, stores it in etcd, controllers reconcile it, the scheduler picks a node, and kubelet on that node makes the Pod real. Everything else is a layer on top of that loop. ([Kubernetes][1])

## A. 10 second summary of Kubernetes

Kubernetes is a control system for containers. You declare what you want, such as 3 Pods, a stable Service endpoint, some storage, and some policy. Kubernetes then keeps correcting drift until reality matches the spec. Pods run on Nodes, Services give stable networking, controllers like Deployment or StatefulSet keep things alive, and storage, security, scaling, and observability are all separate concerns. ([Kubernetes][2])

## B. Main concepts grouped by category

### 1. Cluster anatomy first

| Concept            | Brutally simple definition                                                      | Why it exists                                                                       | Interview trap                                                                         |
| ------------------ | ------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------- |
| API Server         | The front door of the cluster. Every request goes through it. ([Kubernetes][1]) | It validates, authenticates, authorizes, and stores object state. ([Kubernetes][1]) | People think kubectl talks to nodes. It talks to the API server.                       |
| etcd               | The cluster database. It stores Kubernetes state. ([Kubernetes][3])             | Without it, the control plane forgets desired state. ([Kubernetes][3])              | It is not a workload database. It is the cluster brain memory.                         |
| Controller Manager | The set of control loops that keep state converged. ([Kubernetes][4])           | It recreates Pods, updates ReplicaSets, and reconciles drift. ([Kubernetes][4])     | It does not run your app. It runs reconciliation logic.                                |
| Scheduler          | Picks the node for a Pod that has no node yet. ([Kubernetes][5])                | It matches Pod needs to node capacity and constraints. ([Kubernetes][5])            | Scheduling is not placement by round robin. Constraints matter.                        |
| kubelet            | The node agent. It makes sure Pods on a node actually run. ([Kubernetes][6])    | It starts containers, reports health, and enforces limits. ([Kubernetes][7])        | kubelet is on the node, not in the control plane.                                      |
| kube-proxy         | The node side network proxy for Services. ([Kubernetes][8])                     | It implements Service forwarding rules on each node. ([Kubernetes][8])              | It is not Ingress. It is not a load balancer.                                          |
| Node               | A machine that runs Pods. ([Kubernetes][7])                                     | Kubernetes needs worker machines to actually execute containers. ([Kubernetes][7])  | Pods run on nodes, not on the cluster abstractly.                                      |
| NodePool           | A provider level grouping of Nodes. Not a core Kubernetes API object.           | It exists so cloud platforms can scale and manage node groups separately.           | Confusing it with Node causes bad design. Kubernetes talks about Nodes, not NodePools. |

#### YAML or kubectl example

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: demo-pod
spec:
  containers:
    - name: app
      image: nginx:1.27
```

```bash
kubectl get pods
kubectl describe pod demo-pod
kubectl logs demo-pod
```

#### Interview question

What is the difference between the API server and kubelet?

Model answer: the API server stores and validates desired state for the whole cluster, while kubelet is the node agent that turns Pod specs into running containers on one specific node. ([Kubernetes][1])

---

### 2. Workload objects

| Concept           | Brutally simple definition                                                                        | Why it exists                                                                                 | Interview trap                                                        |
| ----------------- | ------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------- | --------------------------------------------------------------------- |
| Pod               | The smallest deployable unit. One or more containers share network and storage. ([Kubernetes][9]) | Kubernetes schedules Pods, not bare containers. ([Kubernetes][9])                             | Pod is not a container. A Pod can hold multiple containers.           |
| Container         | A packaged process plus its dependencies. ([Kubernetes][10])                                      | It gives repeatable runtime behavior. ([Kubernetes][10])                                      | People treat a container as a VM. It is not.                          |
| Init Container    | Runs to completion before app containers start. ([Kubernetes][11])                                | Useful for setup, migration, or waiting for dependencies. ([Kubernetes][12])                  | It is not a sidecar. It exits before the app begins.                  |
| Sidecar Container | A secondary container in the same Pod that extends the app. Stable in v1.33. ([Kubernetes][13])   | Used for logging, proxying, syncing, and helpers without app code changes. ([Kubernetes][13]) | Sidecar is not an init container. It keeps running.                   |
| Deployment        | Manages stateless Pods and ReplicaSets with rolling updates. ([Kubernetes][14])                   | It is the default controller for web apps and APIs. ([Kubernetes][14])                        | Do not use Deployment for sticky identity plus per Pod storage.       |
| ReplicaSet        | Keeps a stable count of identical Pods. Usually owned by a Deployment. ([Kubernetes][15])         | It preserves replica count and supports scaling. ([Kubernetes][15])                           | People create ReplicaSets directly when they really want Deployments. |
| StatefulSet       | Manages stateful Pods with stable identity and sticky storage. ([Kubernetes][16])                 | Databases and consensus systems need ordered identity and storage. ([Kubernetes][16])         | All Pods are not interchangeable. Stateful workloads are different.   |
| DaemonSet         | Ensures one copy of a Pod on each node or selected nodes. ([Kubernetes][17])                      | Used for node agents like log collectors or monitoring agents. ([Kubernetes][17])             | It scales with nodes, not replicas.                                   |
| Job               | Runs work to completion and stops. ([Kubernetes][18])                                             | Batch, migrations, imports, and one off tasks. ([Kubernetes][18])                             | Do not use Job for a long running service.                            |
| CronJob           | Creates Jobs on a schedule. ([Kubernetes][19])                                                    | Useful for backups, reports, cleanup, and periodic tasks. ([Kubernetes][19])                  | CronJob schedules Jobs, not Pods directly.                            |

#### Realistic YAML examples

Deployment:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: checkout-api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: checkout-api
  template:
    metadata:
      labels:
        app: checkout-api
    spec:
      containers:
        - name: app
          image: ghcr.io/company/checkout-api:1.4.2
          ports:
            - containerPort: 8080
          readinessProbe:
            httpGet:
              path: /ready
              port: 8080
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
```

StatefulSet:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres-hl
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
          image: postgres:16
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 20Gi
```

DaemonSet:

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-logger
spec:
  selector:
    matchLabels:
      app: node-logger
  template:
    metadata:
      labels:
        app: node-logger
    spec:
      containers:
        - name: agent
          image: fluent/fluent-bit:2.2
```

Job:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migrate
spec:
  template:
    spec:
      restartPolicy: OnFailure
      containers:
        - name: migrate
          image: ghcr.io/company/migrate:1.0.0
          command: ["sh", "-c", "alembic upgrade head"]
```

CronJob:

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: nightly-backup
spec:
  schedule: "0 2 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: backup
              image: ghcr.io/company/backup:2.1.0
              command: ["sh", "-c", "./backup.sh"]
```

#### Interview question

When would you choose StatefulSet over Deployment?

Model answer: choose StatefulSet when each Pod needs a stable name, stable storage, or ordered rollout, such as databases or consensus systems. Use Deployment for stateless replicas that can be replaced freely. ([Kubernetes][16])

---

### 3. Networking, service discovery, and config

| Concept               | Brutally simple definition                                                                                            | Why it exists                                                                                                      | Interview trap                                                                        |
| --------------------- | --------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------- |
| Service               | A stable network endpoint for one or more Pods. ([Kubernetes][20])                                                    | Pods come and go, so you need a stable front door. ([Kubernetes][20])                                              | A Service does not create Pods. It points to them.                                    |
| ClusterIP             | Default Service type. Gives an internal cluster IP only. ([Kubernetes][20])                                           | Best for internal service to service traffic. ([Kubernetes][20])                                                   | It is internal only unless combined with another exposure method.                     |
| NodePort              | Exposes a Service on a static port on every node. ([Kubernetes][20])                                                  | Simple external access, mostly for dev or basic exposure. ([Kubernetes][20])                                       | People confuse NodePort with a cloud load balancer.                                   |
| LoadBalancer          | Asks the cloud provider for an external load balancer. ([Kubernetes][20])                                             | Standard way to expose services externally in cloud clusters. ([Kubernetes][20])                                   | Depends on cloud provider support. It is not pure Kubernetes magic.                   |
| Headless Service      | A Service with `clusterIP: None`, so no virtual IP is allocated. ([Kubernetes][20])                                   | Lets clients talk directly to Pod IPs, often for StatefulSets or custom discovery. ([Kubernetes][20])              | Headless does not mean no Service object. It still exists for discovery.              |
| Ingress               | HTTP or HTTPS rule based external entry point to Services. ([Kubernetes][21])                                         | Gives host and path routing, TLS termination, and virtual hosting. ([Kubernetes][21])                              | Ingress is frozen. Gateway API is the current preferred direction. ([Kubernetes][21]) |
| Ingress Controller    | The controller that actually implements Ingress rules. ([Kubernetes][22])                                             | Ingress is only an API object. A controller does the traffic handling. ([Kubernetes][22])                          | Many people create Ingress and forget the controller. Nothing works without it.       |
| Service Mesh          | A traffic layer that adds security, observability, and traffic control. ([Istio][23])                                 | Useful when you need mTLS, retries, traffic shaping, and rich telemetry across services. ([Istio][23])             | It is not a replacement for Service. It sits above Service.                           |
| DNS and CoreDNS       | Kubernetes creates DNS records for Services and Pods, and CoreDNS is the common cluster DNS addon. ([Kubernetes][24]) | It lets apps discover other apps by name instead of IP. ([Kubernetes][24])                                         | DNS failures are often really Service or namespace mistakes.                          |
| kube-proxy            | The node side proxy that translates Service behavior into forwarding rules. ([Kubernetes][8])                         | It makes cluster IPs and Service ports work on each node. ([Kubernetes][8])                                        | kube-proxy is part of Service networking, not ingress routing.                        |
| Environment Variables | A way to pass config into a container. `env` and `envFrom` can read ConfigMaps or Secrets. ([Kubernetes][25])         | Good for small values like endpoints, flags, and feature switches. ([Kubernetes][26])                              | Do not bake environment specific config into images.                                  |
| Downward API          | Exposes Pod and container fields through env vars or files. ([Kubernetes][27])                                        | Lets an app know its own name, namespace, labels, or resource settings without calling the API. ([Kubernetes][27]) | Not every API field is available.                                                     |
| ConfigMap             | Stores non confidential key value config. ([Kubernetes][26])                                                          | Keeps config separate from images. ([Kubernetes][26])                                                              | People sometimes put secrets in ConfigMaps. That is wrong.                            |
| Secret                | Stores small amounts of sensitive data. ([Kubernetes][28])                                                            | For passwords, tokens, keys, and registry auth. ([Kubernetes][28])                                                 | Secret is not perfect protection. It is still an object in the API.                   |

#### YAML examples

Service and headless Service:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: checkout-api
spec:
  selector:
    app: checkout-api
  ports:
    - port: 80
      targetPort: 8080
  type: ClusterIP
---
apiVersion: v1
kind: Service
metadata:
  name: postgres-hl
spec:
  clusterIP: None
  selector:
    app: postgres
  ports:
    - port: 5432
      targetPort: 5432
```

Ingress:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: checkout
spec:
  ingressClassName: nginx
  rules:
    - host: checkout.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: checkout-api
                port:
                  number: 80
```

ConfigMap plus Secret plus env:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: checkout-config
data:
  LOG_LEVEL: info
  PAYMENT_TIMEOUT_MS: "3000"
---
apiVersion: v1
kind: Secret
metadata:
  name: checkout-secret
type: Opaque
stringData:
  DATABASE_URL: postgres://user:pass@postgres:5432/shop
---
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
    - name: app
      image: ghcr.io/company/app:1.0.0
      envFrom:
        - configMapRef:
            name: checkout-config
        - secretRef:
            name: checkout-secret
```

#### Interview question

Ingress vs Service?

Model answer: Service gives stable in cluster networking to Pods. Ingress is the HTTP or HTTPS entry point from outside the cluster and needs an Ingress Controller. For internal service to service traffic, use Service. For HTTP routing from outside, use Ingress or, preferably now, Gateway API. ([Kubernetes][20])

---

### 4. Storage and persistence

| Concept               | Brutally simple definition                                                                                  | Why it exists                                                                                   | Interview trap                                                |
| --------------------- | ----------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------- | ------------------------------------------------------------- |
| Volume                | A directory that containers in a Pod can share. ([Kubernetes][29])                                          | Solves shared data and data durability inside a Pod. ([Kubernetes][29])                         | A volume is not automatically persistent.                     |
| PersistentVolume      | Cluster storage resource, usually provisioned by admin or dynamically. ([Kubernetes][30])                   | It decouples storage lifecycle from Pod lifecycle. ([Kubernetes][29])                           | Deleting a Pod does not delete a PV by default.               |
| PersistentVolumeClaim | A user request for storage. The control plane binds it to a matching PV. ([Kubernetes][31])                 | Lets app owners ask for storage without knowing the backing storage details. ([Kubernetes][31]) | A PVC is a request, not the actual disk.                      |
| StorageClass          | A template for classes of storage with parameters and provisioner. ([Kubernetes][32])                       | Enables dynamic provisioning and storage policies. ([Kubernetes][33])                           | Different clouds map StorageClass differently.                |
| CSI                   | The Container Storage Interface, the standard plugin model for external storage drivers. ([Kubernetes][34]) | Lets storage vendors integrate cleanly with Kubernetes. ([Kubernetes][34])                      | CSI is a driver model, not a storage type.                    |
| Ephemeral Volume      | Storage that lives and dies with the Pod. ([Kubernetes][34])                                                | Good for cache, scratch space, config, and temporary data. ([Kubernetes][34])                   | Do not use ephemeral storage for data you need after restart. |

#### YAML examples

PVC:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pg-data
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: fast-ssd
  resources:
    requests:
      storage: 20Gi
```

StorageClass:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: csi.example.com
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
parameters:
  type: ssd
```

Pod using PVC:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: db
spec:
  containers:
    - name: postgres
      image: postgres:16
      volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: pg-data
```

#### Interview question

What is the relationship between PV, PVC, and StorageClass?

Model answer: PVC is the request from the user, PV is the actual storage resource, and StorageClass tells Kubernetes how to dynamically provision or select storage. The common production pattern is PVC plus StorageClass, not manually creating PVs unless you need special control. ([Kubernetes][31])

---

### 5. Scheduling, placement, and scaling

| Concept                      | Brutally simple definition                                                                     | Why it exists                                                                              | Interview trap                                                        |
| ---------------------------- | ---------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------ | --------------------------------------------------------------------- |
| Scheduler                    | Matches Pods to Nodes. ([Kubernetes][5])                                                       | It decides placement based on constraints and capacity. ([Kubernetes][5])                  | Scheduling is about finding a valid node, not starting the container. |
| NodePool                     | Provider level group of nodes that can scale together.                                         | Useful for separating general purpose, GPU, or spot capacity.                              | It is not a native K8s object. Always ask which provider defines it.  |
| Taints and Tolerations       | Taints repel Pods from Nodes, tolerations let Pods accept them. ([Kubernetes][35])             | Used for dedicated nodes, system nodes, GPUs, or noisy workloads. ([Kubernetes][35])       | Toleration allows scheduling, it does not force it.                   |
| Affinity and Anti affinity   | Rules that attract or repel Pods from nodes or other Pods. ([Kubernetes][36])                  | Used for zone spreading, co location, and fault isolation. ([Kubernetes][36])              | People mix up node affinity with pod anti affinity.                   |
| Resource Requests and Limits | Requests are what the scheduler reserves, limits are what kubelet enforces. ([Kubernetes][37]) | Prevents overcommit surprises and improves placement accuracy. ([Kubernetes][37])          | Request is for scheduling. Limit is for runtime enforcement.          |
| QoS Classes                  | Guaranteed, Burstable, BestEffort. Derived from requests and limits. ([Kubernetes][38])        | Used during node pressure eviction. ([Kubernetes][38])                                     | Guaranteed requires all containers to have equal requests and limits. |
| HPA                          | Automatically changes Pod replicas based on metrics. ([Kubernetes][39])                        | Great for handling traffic spikes without touching pod size. ([Kubernetes][39])            | HPA scales Pods, not nodes.                                           |
| VPA                          | Automatically adjusts Pod resource requests and limits. ([Kubernetes][40])                     | Useful when you want right sized CPU and memory. ([Kubernetes][40])                        | VPA is not a replacement for HPA in many apps.                        |
| Cluster Autoscaler           | Adds or removes nodes when Pods cannot be scheduled or nodes are underused. ([GitHub][41])     | Keeps the cluster itself elastic. ([Kubernetes][42])                                       | It scales nodes, not Pods.                                            |
| Metrics Server               | Fetches resource metrics from kubelets and exposes them for HPA and VPA. ([Kubernetes][43])    | Without it, many autoscaling workflows cannot see CPU and memory usage. ([Kubernetes][43]) | Metrics Server is usually not installed by default.                   |

#### YAML example

HPA:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: checkout-api
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: checkout-api
  minReplicas: 3
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

#### Interview question

HPA vs VPA vs Cluster Autoscaler?

Model answer: HPA changes replica count, VPA changes Pod resource sizing, and Cluster Autoscaler changes node count. Use HPA for traffic, VPA for right sizing, and Cluster Autoscaler for capacity. ([Kubernetes][39])

---

### 6. Security, identity, and policy

| Concept                | Brutally simple definition                                                             | Why it exists                                                                        | Interview trap                                                                                |
| ---------------------- | -------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------ | --------------------------------------------------------------------------------------------- |
| Namespace              | Logical isolation inside one cluster. ([Kubernetes][44])                               | It separates names, quotas, policy, and team boundaries. ([Kubernetes][44])          | Namespace is not a hard security boundary by itself.                                          |
| RBAC                   | Role based access control for Kubernetes objects. ([Kubernetes][45])                   | It limits who can do what on which resources. ([Kubernetes][45])                     | RBAC is not the same as pod security.                                                         |
| Role                   | Namespaced permission set. ([Kubernetes][45])                                          | Grants access only inside one namespace. ([Kubernetes][45])                          | Do not use Role when cluster scoped access is required.                                       |
| ClusterRole            | Cluster wide permission set. ([Kubernetes][45])                                        | Needed for cluster scoped resources like Nodes or PVs. ([Kubernetes][44])            | It is not automatically powerful. Binding decides scope.                                      |
| RoleBinding            | Connects a Role or ClusterRole to subjects within a namespace. ([Kubernetes][45])      | Gives a user, group, or ServiceAccount the permissions. ([Kubernetes][45])           | Binding scope matters more than the role name alone.                                          |
| ClusterRoleBinding     | Connects a ClusterRole to subjects cluster wide. ([Kubernetes][45])                    | Needed for cluster admin style access. ([Kubernetes][45])                            | Easy way to over grant permissions if you are careless.                                       |
| ServiceAccount         | Identity for processes running in a Pod. ([Kubernetes][46])                            | Lets workloads talk to the API server with a controlled identity. ([Kubernetes][46]) | It is not a human user account.                                                               |
| Pod Security Standards | Three levels, privileged, baseline, restricted. ([Kubernetes][47])                     | Standard way to enforce pod hardening. ([Kubernetes][48])                            | PSP is legacy. Current preferred practice is Pod Security Admission. ([Kubernetes][48])       |
| NetworkPolicy          | Rules for how Pods can talk to each other and to the outside. ([Kubernetes][49])       | Default deny or segmented networking in multi tenant clusters. ([Kubernetes][49])    | Works only if your CNI enforces it.                                                           |
| Admission Controller   | Gate that intercepts API writes before persistence. ([Kubernetes][1])                  | Used for defaulting, validation, mutation, and policy. ([Kubernetes][1])             | It does not block reads. Only writes and some custom verbs.                                   |
| OPA Gatekeeper         | Policy controller that enforces CRD based policies with OPA. ([Open Policy Agent][50]) | Used for policy as code, audit, and admission enforcement. ([Open Policy Agent][51]) | For simple validation, built in Validating Admission Policy may be enough. ([Kubernetes][52]) |

#### YAML example

Namespace plus RBAC plus ServiceAccount:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: payments
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: payments-app
  namespace: payments
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: payments-reader
  namespace: payments
rules:
  - apiGroups: [""]
    resources: ["pods", "services"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: payments-reader-binding
  namespace: payments
subjects:
  - kind: ServiceAccount
    name: payments-app
    namespace: payments
roleRef:
  kind: Role
  name: payments-reader
  apiGroup: rbac.authorization.k8s.io
```

NetworkPolicy:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-api-to-db
  namespace: payments
spec:
  podSelector:
    matchLabels:
      app: db
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

#### Interview question

Namespace vs RBAC vs Pod Security Admission?

Model answer: namespace separates objects and is the scope for many policies, RBAC decides who can do what to resources, and Pod Security Admission enforces pod hardening rules at namespace level when pods are created. They solve different problems. ([Kubernetes][44])

---

### 7. Observability and debugging

| Concept          | Brutally simple definition                                                                     | Why it exists                                                              | Interview trap                                                                         |
| ---------------- | ---------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------- | -------------------------------------------------------------------------------------- |
| Logs             | Container output collected by the kubelet and exposed through kubectl logs. ([Kubernetes][53]) | Fastest way to see what the app itself said. ([Kubernetes][54])            | Only the latest rotated log file is available through kubectl logs. ([Kubernetes][53]) |
| Events           | Time ordered record of what Kubernetes did to objects. ([Kubernetes][55])                      | Great for scheduling, pull, mount, and policy failures. ([Kubernetes][56]) | Events are not the same as app logs.                                                   |
| Metrics Server   | Supplies resource metrics for HPA, VPA, and kubectl top. ([Kubernetes][43])                    | Lets autoscalers and humans see CPU and memory usage. ([Kubernetes][43])   | It is not a full observability stack.                                                  |
| Prometheus       | Metrics scraping and alerting system. ([Prometheus][57])                                       | Industry standard for time series and alerting. ([Prometheus][57])         | Prometheus stores metrics. Grafana visualizes them.                                    |
| Grafana          | Dashboard and visualization layer for metrics, logs, and traces. ([Grafana Labs][58])          | Lets humans inspect trends and incident data fast. ([Grafana Labs][59])    | Grafana is not the database.                                                           |
| Liveness Probe   | Tells Kubernetes when to restart a container. ([Kubernetes][60])                               | Fixes dead or hung processes automatically. ([Kubernetes][60])             | Do not use liveness for startup delay problems.                                        |
| Readiness Probe  | Tells Kubernetes when a container is ready to receive traffic. ([Kubernetes][60])              | Prevents sending traffic too early. ([Kubernetes][60])                     | Readiness does not restart the container.                                              |
| Startup Probe    | Gives a slow starting app time to boot before other probes begin. ([Kubernetes][61])           | Useful for apps with long initialization. ([Kubernetes][61])               | If startup probe succeeds, liveness and readiness start running. ([Kubernetes][61])    |
| kubectl describe | Shows detailed object state and related events. ([Kubernetes][56])                             | First command for almost every broken Pod or Service. ([Kubernetes][56])   | It is human readable, not for scripts. ([Kubernetes][62])                              |
| kubectl logs     | Prints logs for a Pod or resource. ([Kubernetes][54])                                          | Primary way to inspect application output. ([Kubernetes][54])              | Use `--previous` for the crashed container instance. ([Kubernetes][54])                |

#### YAML example

Probes:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: probe-demo
spec:
  containers:
    - name: app
      image: ghcr.io/company/app:1.0.0
      ports:
        - containerPort: 8080
      startupProbe:
        httpGet:
          path: /startup
          port: 8080
        failureThreshold: 30
        periodSeconds: 2
      readinessProbe:
        httpGet:
          path: /ready
          port: 8080
      livenessProbe:
        httpGet:
          path: /health
          port: 8080
```

#### Interview question

Why do you need both readiness and liveness?

Model answer: readiness controls traffic. Liveness controls restart. Readiness keeps broken or warming Pods out of Service endpoints, while liveness repairs dead Pods. Startup probe is for slow booting apps so liveness does not kill them too early. ([Kubernetes][60])

---

### 8. Platform, packaging, delivery, and extensibility

| Concept                  | Brutally simple definition                                                             | Why it exists                                                                                             | Interview trap                                                       |
| ------------------------ | -------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------- |
| Helm                     | Kubernetes package manager with charts. ([Helm][63])                                   | Bundles and versions whole applications. ([Helm][64])                                                     | Helm is packaging, not runtime orchestration.                        |
| Kustomize                | Tool for composing and customizing Kubernetes manifests. ([Kubernetes][65])            | Great for overlays, environment tweaks, and base reuse. ([Kubernetes][65])                                | Kustomize is not templating in the Helm sense.                       |
| CRD                      | Extends the Kubernetes API with your own resource types. ([Kubernetes][66])            | Needed when core resources are not enough. ([Kubernetes][66])                                             | CRD adds an API type. It does not add behavior by itself.            |
| Operator                 | Controller plus CRD pattern for managing apps the Kubernetes way. ([Kubernetes][67])   | Encodes day 2 operations for complex systems. ([Kubernetes][67])                                          | Operators are not just fancy manifests. They run control loops.      |
| GitOps                   | Declare desired state in Git and sync clusters to it. ([fluxcd.io][68])                | Gives auditability, rollback, and declarative deployment. ([fluxcd.io][68])                               | GitOps is a process, not a tool.                                     |
| Argo CD                  | Controller that compares live state to Git desired state and syncs it. ([Argo CD][69]) | Popular GitOps CD system for Kubernetes. ([Argo CD][69])                                                  | OutOfSync means cluster drift from Git.                              |
| Flux                     | GitOps CD solution that keeps clusters in sync with config sources. ([fluxcd.io][70])  | Automated continuous delivery for Kubernetes. ([fluxcd.io][70])                                           | Flux is a set of controllers, not just one binary.                   |
| Istio                    | Service mesh for security, observability, and traffic management. ([Istio][23])        | Needed for advanced traffic shaping and mTLS. ([Istio][23])                                               | Istio is not required for basic Kubernetes networking.               |
| Linkerd                  | Lightweight service mesh for security, observability, and reliability. ([Linkerd][71]) | Simpler mesh for teams that want less complexity. ([Linkerd][72])                                         | Service mesh is optional infrastructure.                             |
| Knative                  | Kubernetes based serverless platform for modern workloads. ([Knative][73])             | Useful for request driven and event driven apps, including scale to zero style workloads. ([Knative][74]) | Knative is a higher level platform on top of Kubernetes.             |
| Rolling Update           | Gradual replacement of old Pods with new ones. ([Kubernetes][14])                      | Safer default rollout strategy for most web services. ([Kubernetes][14])                                  | Not all apps tolerate partial version mixing.                        |
| Canary Deployment        | Send a small slice of traffic to a new version.                                        | Reduces blast radius for risky releases.                                                                  | Needs traffic splitting support, often from mesh or gateway tooling. |
| Blue Green Deployment    | Run old and new versions side by side, then switch traffic.                            | Fast rollback by flipping traffic back.                                                                   | Costs more because both environments may run at once.                |
| Immutable Infrastructure | Replace rather than mutate running servers or Pods.                                    | Fewer snowflakes, more repeatability.                                                                     | Do not patch live containers by hand and call it a strategy.         |
| Infrastructure as Code   | Infrastructure described in versioned files.                                           | Reproducibility and reviewability.                                                                        | YAML and Terraform are tools, not the principle itself.              |
| CI CD Pipeline           | Automated build, test, scan, and deploy flow.                                          | Makes delivery repeatable and safer.                                                                      | CI builds code. CD moves desired state toward prod.                  |

#### YAML example

Argo style GitOps usually uses Kubernetes manifests plus an Application CRD, but the core idea is the same. The desired state lives in Git, and the controller reconciles it. ([Argo CD][69])

#### Interview question

Helm vs Kustomize vs Operators?

Model answer: Helm packages and templatizes apps, Kustomize overlays and patches existing manifests, and Operators add controller logic for day 2 automation. Helm and Kustomize are about manifest management. Operators are about behavior. ([Helm][63])

---

## C. Side by side comparisons

### Pod vs Container

| Pod                                                                             | Container                                             |
| ------------------------------------------------------------------------------- | ----------------------------------------------------- |
| Smallest deployable unit in Kubernetes. ([Kubernetes][9])                       | Packaged process and dependencies. ([Kubernetes][10]) |
| May hold multiple containers. ([Kubernetes][9])                                 | Runs one workload process.                            |
| Shares network and storage namespace with sibling containers. ([Kubernetes][9]) | One runtime image instance.                           |

### Deployment vs StatefulSet vs DaemonSet vs Job vs CronJob

| Type        | Best for                                                 | Why                                 |
| ----------- | -------------------------------------------------------- | ----------------------------------- |
| Deployment  | Stateless web apps and APIs. ([Kubernetes][14])          | Rolling updates and easy scaling.   |
| StatefulSet | Databases, queues, consensus systems. ([Kubernetes][16]) | Stable identity and stable storage. |
| DaemonSet   | Node agents. ([Kubernetes][17])                          | One Pod per node.                   |
| Job         | One time work. ([Kubernetes][18])                        | Runs to completion.                 |
| CronJob     | Scheduled work. ([Kubernetes][19])                       | Creates Jobs on a schedule.         |

### Service types

| Type         | What it does                                           | When to use                                      |
| ------------ | ------------------------------------------------------ | ------------------------------------------------ |
| ClusterIP    | Internal virtual IP only. ([Kubernetes][20])           | Default for in cluster service to service calls. |
| NodePort     | Exposes a port on every node. ([Kubernetes][20])       | Simple external access or debugging.             |
| LoadBalancer | Provisions a cloud load balancer. ([Kubernetes][20])   | Production external exposure in cloud.           |
| Headless     | No virtual IP. DNS returns Pod IPs. ([Kubernetes][20]) | Stateful identity or direct Pod discovery.       |

### ConfigMap vs Secret

| ConfigMap                                                | Secret                                                            |
| -------------------------------------------------------- | ----------------------------------------------------------------- |
| Non confidential config. ([Kubernetes][26])              | Sensitive data like passwords, tokens, keys. ([Kubernetes][28])   |
| Common for flags and endpoints. ([Kubernetes][25])       | Common for registry auth and DB passwords. ([Kubernetes][75])     |
| Use with env, args, or mounted files. ([Kubernetes][26]) | Same consumption patterns, but more sensitive. ([Kubernetes][25]) |

### HPA vs VPA vs Cluster Autoscaler

| HPA                                      | VPA                                                 | Cluster Autoscaler                                                        |
| ---------------------------------------- | --------------------------------------------------- | ------------------------------------------------------------------------- |
| Scales replica count. ([Kubernetes][39]) | Adjusts Pod requests and limits. ([Kubernetes][40]) | Scales node count. ([GitHub][76])                                         |
| Good for traffic changes.                | Good for right sizing.                              | Good for capacity changes.                                                |
| Needs metrics. ([Kubernetes][43])        | Needs metrics. ([Kubernetes][40])                   | Needed when Pods are pending due to lack of node capacity. ([GitHub][76]) |

### Probe types

| Probe     | What it answers                                        | What happens on failure                                         |
| --------- | ------------------------------------------------------ | --------------------------------------------------------------- |
| Liveness  | Is the container still healthy. ([Kubernetes][60])     | Kubernetes restarts the container.                              |
| Readiness | Is the container ready for traffic. ([Kubernetes][60]) | Kubernetes removes it from Service endpoints.                   |
| Startup   | Has the app finished booting. ([Kubernetes][61])       | Liveness and readiness wait until it passes. ([Kubernetes][61]) |

### Ingress vs Service

| Service                                                                                                                                    | Ingress                                                                 |
| ------------------------------------------------------------------------------------------------------------------------------------------ | ----------------------------------------------------------------------- |
| Gives stable networking to Pods. ([Kubernetes][20])                                                                                        | Gives HTTP or HTTPS entry from outside. ([Kubernetes][21])              |
| Works for internal traffic and all protocols supported by Service types. ([Kubernetes][20])                                                | Best for HTTP routing, host and path rules. ([Kubernetes][21])          |
| Does not need an extra controller to exist as an object, but traffic behavior is implemented by kube-proxy or providers. ([Kubernetes][8]) | Needs an Ingress Controller to actually do the work. ([Kubernetes][22]) |

---

## D. Common errors and how to debug them

Start with `kubectl describe`, then `kubectl events`, then `kubectl logs`. For crashed containers use `kubectl logs --previous`. For DNS problems, use the DNS debugging guide. ([Kubernetes][56])

| Symptom                    | Likely meaning                                                                                                                                                                                                                                                                        | First commands                                                                                | Typical fix                                                                                |
| -------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------ |
| ErrImagePull               | Image could not be pulled, often bad name, auth, registry access, or node compatibility. ([Kubernetes][77])                                                                                                                                                                           | `kubectl describe pod`, `kubectl events` ([Kubernetes][56])                                   | Fix image name, registry auth, imagePullSecret, or node architecture. ([Kubernetes][77])   |
| ImagePullBackOff           | Same root problem as image pull failure, with backoff retry. ([Kubernetes][77])                                                                                                                                                                                                       | `kubectl describe pod` and check Events. ([Kubernetes][78])                                   | Correct registry access or image reference. ([Kubernetes][77])                             |
| CrashLoopBackOff           | Container starts, crashes, then Kubernetes backs off and retries. ([Kubernetes][79])                                                                                                                                                                                                  | `kubectl logs --previous`, `kubectl describe pod`, `kubectl events` ([Kubernetes][54])        | Fix code bugs, bad env, missing config, bad probes, or resource limits. ([Kubernetes][79]) |
| OOMKilled                  | Container or Pod was killed for memory pressure or limit breach. ([Kubernetes][80])                                                                                                                                                                                                   | `kubectl describe pod`, inspect requests and limits. ([Kubernetes][81])                       | Increase memory limit, lower memory use, or fix leaks. ([Kubernetes][81])                  |
| CreateContainerConfigError | The Pod spec points to bad or missing configuration. This is usually a missing Secret, ConfigMap, bad env reference, or invalid volume reference. This is an operational inference from how env, Secret, ConfigMap, and volume setup works before container start. ([Kubernetes][25]) | `kubectl describe pod`, inspect Events, verify referenced objects exist. ([Kubernetes][56])   | Create the missing object or fix the reference.                                            |
| Pending                    | Pod is accepted but not yet scheduled or not yet fully set up. ([Kubernetes][79])                                                                                                                                                                                                     | `kubectl describe pod`, inspect scheduler events. ([Kubernetes][56])                          | Add capacity, relax affinity, adjust taints, or lower requests. ([Kubernetes][37])         |
| ContainerCreating          | Pod is accepted but still pulling image or setting up runtime state. This is an inference from Pod Pending, image pull behavior, and volume setup. ([Kubernetes][79])                                                                                                                 | `kubectl describe pod`, `kubectl events` ([Kubernetes][56])                                   | Wait for image pull, check registry access, or fix volume mount issues.                    |
| Terminating                | Pod is in graceful shutdown. ([Kubernetes][79])                                                                                                                                                                                                                                       | `kubectl describe pod`, check finalizers and grace period. ([Kubernetes][56])                 | Reduce shutdown time, clear blockers, or force delete only when safe. ([Kubernetes][79])   |
| Evicted                    | Kubelet removed the Pod to reclaim node resources. ([Kubernetes][80])                                                                                                                                                                                                                 | `kubectl describe pod`, `kubectl describe node`, inspect pressure and QoS. ([Kubernetes][56]) | Lower usage, set requests and limits, or add capacity. ([Kubernetes][81])                  |

---

## E. Production examples

1. A payments API runs as a Deployment with 3 replicas, a ClusterIP Service, readiness and liveness probes, HPA on CPU, and ConfigMap plus Secret for config. This is the common stateless web pattern. ([Kubernetes][14])

2. A PostgreSQL or Kafka style backend runs as a StatefulSet with volumeClaimTemplates and a headless Service so each Pod has stable identity and its own storage. ([Kubernetes][16])

3. A log shipper or node exporter runs as a DaemonSet so every node reports logs or metrics. ([Kubernetes][17])

4. A nightly ledger reconciliation runs as a CronJob, which creates Jobs on a schedule. ([Kubernetes][19])

5. A multi team platform uses Namespaces, RBAC, Pod Security Admission, and NetworkPolicy to separate teams and reduce blast radius. ([Kubernetes][44])

6. A platform team ships releases with Helm or Kustomize, manages drift with Argo CD or Flux, and uses Istio or Linkerd for canary traffic shifting and mTLS. ([Helm][63])

---

## F. Interview questions and answers

### Workloads

Question: Why does Kubernetes use Pods instead of containers directly?

Answer: Pods are the smallest deployable unit because Kubernetes needs a shared network and storage boundary, not just an isolated process. That is why one Pod can contain sidecars and init containers. ([Kubernetes][9])

### Networking

Question: Why is a headless Service useful?

Answer: A headless Service removes the virtual IP and gives direct Pod endpoints through DNS. That is useful when clients need to talk to specific Pods, usually for stateful systems. ([Kubernetes][20])

### Storage

Question: Why not mount hostPath everywhere?

Answer: hostPath ties data to a specific node and breaks portability. PV, PVC, and StorageClass give Kubernetes a portable storage abstraction with dynamic provisioning. ([Kubernetes][29])

### Scheduling and autoscaling

Question: Why does a Pod stay Pending?

Answer: Because the scheduler has not found a valid node yet, often due to lack of capacity, strong affinity rules, taints, or storage topology constraints. ([Kubernetes][79])

### Security

Question: What is the difference between RBAC and Pod Security Admission?

Answer: RBAC controls who can do what to Kubernetes resources. Pod Security Admission controls what a Pod spec is allowed to look like in a namespace. They solve different layers of the problem. ([Kubernetes][45])

### Delivery

Question: What is the real difference between Helm and GitOps?

Answer: Helm is a packaging tool. GitOps is an operating model where Git is the source of truth and a controller keeps the cluster in sync with it. Argo CD and Flux are GitOps engines. ([Helm][63])

---

## G. Final cheat sheet

1. Pod is the unit Kubernetes schedules. Container is the unit the runtime runs. ([Kubernetes][9])
2. Deployment is for stateless replicas. StatefulSet is for sticky identity and storage. DaemonSet is one per node. Job is run to completion. CronJob is scheduled Job creation. ([Kubernetes][14])
3. Service gives stable networking. Ingress gives HTTP entry. Ingress Controller does the actual work. Gateway API is the newer preferred direction. ([Kubernetes][20])
4. ConfigMap is non secret config. Secret is sensitive config. Downward API exposes Pod facts to containers. ([Kubernetes][26])
5. PVC asks for storage. PV is storage. StorageClass decides how it is provisioned. ([Kubernetes][31])
6. Requests help scheduling. Limits help runtime enforcement. QoS decides eviction order under pressure. ([Kubernetes][37])
7. HPA changes replicas. VPA changes Pod size. Cluster Autoscaler changes node count. ([Kubernetes][39])
8. Logs show what the app said. Events show what Kubernetes did. `describe` shows both in one place. ([Kubernetes][54])
9. CrashLoopBackOff is app crash or bad probe or bad config. ImagePullBackOff is image access or registry failure. OOMKilled is memory pressure or limit breach. ([Kubernetes][79])
10. Namespace is isolation by convention. RBAC is permission. Pod Security Admission is pod hardening. NetworkPolicy is traffic control. ([Kubernetes][44])

Memory trick:

Pod is the house. Container is the person. Service is the address. Deployment is the property manager. StatefulSet is the house with fixed room numbers. DaemonSet is one guard per building. Job is one task. CronJob is a scheduled task. PVC is a storage request. HPA is more workers. VPA is bigger workers. Cluster Autoscaler is more buildings. ([Kubernetes][9])