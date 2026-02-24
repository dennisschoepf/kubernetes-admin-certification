# Introduction to Kubernetes

## Container Orchestration

- Containers run container images, not applications, but applications can be packaged as container images.
- Bottom-Up: Hardware -> Operating System -> Container Runtime -> Container (/bin library -> App)
- Orchestrators handle fault-tolerance, scalability, resource usage, discovery, accessibility and updates/rollbacks for containers

### Why?

- To group hosts into a cluster (distributed system, more fault-tolerant)
- To schedule containers to run on hosts based on availability
- To allow for containers to communicate with each other regardless of the host
- To bind storage to containers
- To group similar containers, and add load-balancers, ... to them
- To manage and optimize resource usage and enact secure access to them and the containers

## Kubernetes Features

- Automatic Bin Packing (Scheduling containers based on resource needs to ensure availability)
- Extensibility (Custom features can be added without modifying upstream)
- Self-Healing (Automatic restarts/reschedules, health checks, routing away from unresponsive containers)
- Horizontal scaling (Scale based on CPU other metrics)
- Service discovery (Containers get IPs (4 and 6) and K8s assings DNS names to set of containers for load-balancing)
- Automated rollouts/rollbacks
- Secret and configuration management
- Storage Orchestration (K8s mount software-defined-storage from a variety of storage providers)
- Batch-Execution (long running jobs)
- Can be deployed to many envs (VMs, bare metal, cloud) and has a modular architecture and plugin system

## Architecture

- A cluster consisting of one or more control plane nodes and one or more worker nodes

### Control Plane nodes

- Provides a running environment for control plane agents (responsible for manging the state of the cluster)
- Can be accessed via CLI, Web UI and API
- Control plane has to be running, if it goes down that means downtime and service disruption
- Replicas in HA (high-availability) mode can be added to ensure running control plane
  - Main Node actively manages the cluster, replicas are in sync and take over if the main node fails
- Cluster state is saved to a distributed key-value store (either on the control plane node = stacked or external)

#### Components

- `kube-apiserver`
  - Only component that talks to the key-value-store
  - Intercepts calls from users, admins, devs, operators, ... then validates and processes them
  - Can scale horizontally but also allows for secondary API servers and rules to route requests to them
- `kube-scheduler`
  - Assigns workload objects (e.g. pods) to (mostly) worker nodes
  - Can act based on requirements (e.g scheduling workload to a node that has an SSD if required)
  - Takes into account: data locality, affinity, toleration, topology
  - Gathers all cluster data and filters nodes to select single node that should host the new workload
  - Communicates outcome back to the API server
  - Highly configurable with policies, plugins and profiles and additional schedulers are supported
- Controller Managers
  - Run operator processes to "regulate" the state of the cluster
  - Are watch-loop processes continously comparing desired state to current state of the cluster
  - If desired state != current state the controller manger takes action
  - Can interact with the underlying infrastructure
- Key Value Store `etcd`
  - Strongly consistent, distributed KV-data-store, Append-Only
  - In a stacked HA scenario there is a stacked etcd cluster with replications of the data and this runs on the control plane nodes themselves
  - In an external HA scenario the API servers of the control plane nodes access an external `etcd` cluster
  - Built upon Raft Consensus Algorithm, one node is the leader, all other nodes are followers
  - If the leader fails, follower takes over and becomes the leader
  - Also used to store subnets, ConfigMaps, Secrets, ...

### Worker Nodes

- Environment to run client applications (= microservices running as application containers = Pods)
- Pods are scheduled on worker nodes where they can access compute, memory,storage, networking
- Pod is the smallest scheduling work unit (collection of one or more containers) and can be started, stopped, rescheduled in a single unit of work

#### Components

- Container Runtime
  - Is needed on the actual node so that containers can be run both on control plane nodes (recommended) and worker nodes
  - Options are: CRI-O (lightweight), `containerd`, Docker Engine, ...
- `kubelet` (Node Agent)
  - An Agent that runs on each node, control plane and worker that communicates with the control plane
  - Receives pod definitions and interacts with the container runtime (through grpc and the CRI shim = grpc layer that controls runtime)
  - CRI implements ImageService (handles image related operations) and RuntimeService (handles pod and container related operations)
  - Different shims for different runtimes (`dockershim` or `cri-docked` for Docker, CRI-O for e.g. runC, `cri-containerd` for `containerd`)
- `kube-proxy`
  - Network agent that runs on each node, control plane and worker
  - Responsible for dynamic update and maintenance of networking rules on the node to forward requests to containers within Pods
  - Handles TCP, UDP and SCTP stream forwarding by using `iptables`
- Add-ons
  - DNS, Dashboard, Monitoring, Logging, Device Plugins (GPU, ...)

### Networking

- Container-To-Container within Pods
  - By using kernel virtualization the container runtime creates a network (name)space for each container
  - The namespace can be shared between containers or with the host
  - Containers in a pod can talk to each other via localhost by a "Pause container" that implements a shared namespace for the Pod
- Pod-To-Pod across Nodes
  - Pods can communicate with all other Pods in the cluster (has to be without NAT)
  - Pods are treated as VMs on a network, each Pod gets an IP address
  - Containers share Pod network namespace and port assignment must be coordinated (same as applications in a VM)
  - Containers are integrated into the cluster networking with the Container Network Interface (CNI)
  - CNI actually handles the IP assignment (e.g. with MACvlan, IPvlan, 3rd party, ...)
- External-To-Pod
  - Enabled in the cluster through Services (= set of network routing rules in `iptables` implemented by `kube-proxy`)
  - Applications can become accessible from outside via a virtual IP and dedicated port number

## Installation

- Different configurations are possible (from least to most complex and available):
  - All-in-One Single-Node (everything runs on a single node (not recommeded for production)
  - Single-Control-Plane + Multi-Worker (control plane runs stacked `etcd`)
  - Single-Control-Plane + Single `etcd` + Multi Worker (control plane communicates with external `etcd`)
  - Multi-Control-Plane + Multi-Worker (HA mode, each control plane with a stacked `etcd`)
  - Multi-Control-Plane + Multi-Node `etcd` + Multi-Worker (each control plane has an external `etcd`, these are configured in an `etcd` cluster)
- Decide on the underlying infrastructure
  - Bare Metal, Public/Private/Hybrid Cloud?
  - Which underlying OS?
  - Which networking solution
  - Learning clusters are: `minikube`, `kind`, `microk8s`, `k3s`
  - Production cluster deployment tools are: `kubeadm`, `kubespray`, `kops` - These handle a lot of the bootstrapping tasks and can install onto different kinds of Infra
  - Managed offerings from AWS or others

### Minikube

- Easy local all-in-one or multi-node cluster directly on a workstation
- Responsible for installation of components, cluster bootstraping and teardown by utilizing `kubeadm`
- Sets up a fully functional, non-production cluster (but needs a Type-2 Hypervisor or Container runtime for all features)
- Hypervisor offers an isolated infrastructure for the cluster so that it can be easily teared down
- Can be installed on bare-metal (might lead to permanent Host OS changes) by specifying `--driver`
- Nodes are expected to access the internet

#### Requirements

- For certain hypervisors: VT-x/AMD-v virtualization
- `kubectl` (installed through `minikube` and accessible via `minikube kubectl`)
- Hypervisor or container runtime in the following order (first one is used, unless driver is specified): `docker`, `kvm2`, `podman`, `vmware`, `virtualbox`
- Internet connection on first run (fetch packages, etc.)

#### Working with the cluster

- CLI `kubectl`
  - Can be installed separately and have a different version to Kubernetes (although recommended to keep cloase)
  - Has a configuration file, where server, cluster name user, certificates, ... are stored
- Web UI `Kubernetes Dashboard`
  - Add-On. Uses `metrics-server` to ingest data
- API Server (HTTP)
  - `clusterrole` has to be setup to allow `api-!ccess-root` (RBAC)
  - Either through kubectl proxy or with authenticated (Bearer) requests (`kubectl create token default`, `default` is the service account)

## Building Blocks

- Kubernetes Object Model: Representing different persistent entities in the cluster
- Entities describe:
  - What applications we are running
  - The nodes on which the containers are deployed
  - Application Resource Consumption
  - Policies (Restart/Upgrade/Fault Tolerance, Ingress/Egress, Access Control)
- For each object the intended state is described in `spec`
- Kubernetes manages the `status` section (actual state of the object), control plane does the work of getting to the status
- Other fields are `apiVersion`, `kind`, `metadata`, sometimes `spec` is not but substituted present with `stringData` and `data`

### Nodes

- Virtual identites assigned by Kubernetes to the systems part (VM, Bare-Metal, Containers)
- Each node has a `kubelet` and `kube-proxy` and hosts a container runtime
- Two types of nodes: Control Plane Node and Worker Node
- Node identities are created during bootstrapping
- Can be distributed across different networks

### Namespaces

- Roughly translate to virtual subclusters
- Can be used to achieve multi-tenancy (users, teams, applications, tiers, regions)
- Resource Quotas can be used to limit resources consumed within namespaces

### Pods

- Smallest workload object and the unit of deployment in Kubernetes
- Represents a single instance of an application
- Can consist of one or more containers with volumes that are all on the same host, the same network namespace and have access to the same external storage
- Are ephemeral and can not self-heal (controllers/operators are needed for that)
- The specification of a pod is either in a standalone manifest or in the controllers manifest if managed by that
- Can be ran with `kubectl create/apply -f pod.yaml` or with CLI arguments (save to file with `kubectl run ... > pod.yaml`)
- Important commands: `apply`, `get pods`, `get pod -o yaml/json`, `describe pod`, `delete pod`

### Labels

- Key-Value pairs attached to Kubernetes objects (Pods, ReplicaSets, Nodes, Namespaces, Volumes)
- Used to organize and select subset of objects
- Are used by controllers to logically group together objects
- Label Selectors can be used to select: e.g. `env=dev` or `env in (dev,staging,prod)`

### ReplicaSet

- Implements replication and self healing for pods and supports equality and set-based selectors
- Can scale a number of pods to achieve high availability (either manually or by using an autoscaler)
- Monitors the pods defined for the set and restarts pods if they crashed or otherwise terminated
- Are complemented by the `Deployment` object

### Deployments

- Provide declarative updates for pods and replica sets
- Is part of the control plane node controller manager and ensures that the current state matches the desired state of running the application
- Allows for seamless application updates and rollbacks with a `RollingUpdate` strategy with `rollouts` and `rollbacks`
- Other strategies are available (e.g. `Recreate`)
- Direcly manages ReplicaSets for scaling
- Through pushing updates to the deployment object (e.g. chaging container image) the deployment manages the seamless transition between two ReplicaSets
- Some operations (container image, port, volumes, mount) trigger a new Revision, other (dynamic) operations (scaling, labelling) do not trigger aan update (and therefore do not change the revision number)
- After an update the old replicaset is kept as a prior revision to allow for rolling back the deployment

```
$ kubectl apply -f nginx-deploy.yaml --record
$ kubectl get deployments
$ kubectl get deploy -o wide
$ kubectl scale deploy nginx-deployment --replicas=4
$ kubectl get deploy nginx-deployment -o yaml
$ kubectl get deploy nginx-deployment -o json
$ kubectl describe deploy nginx-deployment
$ kubectl rollout status deploy nginx-deployment
$ kubectl rollout history deploy nginx-deployment
$ kubectl rollout history deploy nginx-deployment --revision=1
$ kubectl set image deploy nginx-deployment nginx=nginx:1.21.5 --record
$ kubectl rollout history deploy nginx-deployment --revision=2
$ kubectl rollout undo deploy nginx-deployment --to-revision=1
$ kubectl get all -l app=nginx -o wide
$ kubectl delete deploy nginx-deployment
$ kubectl get deploy,rs,po -l app=nginx
```

### DaemonSets

- Are operators designed to manage node agents
- Resemble ReplicaSet and Deployment operators when managing multiple pod replicas
- Can enforce a single Pod replica on: per node/all nodes/subset of nodes (ReplicaSet/Deployment have no control over a node)
- Often used for monitoring, storage, networking, proxy daemons to ensure that they run on all nodes and that a type of Pod runs at all times (e.g. `kube-proxy` or other networking node agents)
- When a node is added to the cluster a pod is automatically placed on it (by the DaemonSet itself, not the scheduler - but scheduler can be used)
- On deletion of a daemon set all pod replicas are deleted
- Can be managed with similar commands as deployement (`rollout`, `get ds`, ...)

### Services

- A container in a cluster is not discoverable per default on does not expose ports to the cluster network
- Normally done by simply mapping ports in other container hosts
- In Kubernetes this cant be done, it needs kube-proxy, ip tables, routing rules, dns server and load balancing mechanisms
- This logic is encapsulated in a Service and is the recommended way to expose a containerized application to the outside

## Authentication, Authorization & Admission Control

Each request goes through these in order:

1. Authentication (based on credentials of API requests)
2. Authorization (authorizes API requests submitted by the authenticated user)
3. Admission Controler (software modules that validate/modify user requests)

### Authentication

K8s supports two kinds of users:

- Normal Users (e.g. with user ceritificates, files with username/password, service provider accounts)
- Service Accounts (in cluster process communication to the API server, mostly created automatically)

These auth methods are possible:

- Anonymous = No Auth
- X509 client certificates
- Static token file
- Bootstrap tokens
- Service account tokens
- and openId, custom proxies, webhooks (external)

### Authorization

There are 4 different modes:

1. Node: Specifically for kubelet API requests
2. ABAC: attribute based access control based policies: e.g: `kind: Policy, spec: { user: "bob" }`
3. Webhook: Request third party service
4. RBAC: role based access control `Role` and `ClusterRole`

#### RBAC example

This spec defines a role that can read pods only. A `Role` can only access one namespace. Use `ClusterRole`s to define roles that can act on the cluster as a whole

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
namespace: lfs158
name: pod-reader
rules:
  - apiGroups: [""] # "" indicates the core API group
resources: ["pods"]
verbs: ["get", "watch", "list"]
```

To bind a `ROle` to a user, a `RoleBinding` (or `ClusterRoleBinding` for a `ClusterRole`) has to be created:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pod-read-access
  namespace: lfs158
subjects:
  - kind: User
    name: bob
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

### Admission control

- Controllers used so specify granular access control policies
- Used e.g. for privileged containers, resource quota and other specific data
- To use it the server has to be started with `enable-admission-plugins` and controller names: `--enable-admission-plugins=NamespaceLifecycle,ResourceQuota,PodSecurity,DefaultStorageClass`
- Some are enabled by default, external plugins can be used for this as well and run via webhooks

## Services (In-Depth)

- Used to allow for access to standalone applications
- To access an application a user or another application needs to access a pod
- Pods are ethemeral, so IP adresses cannot be statically allocated
- Services logically group pods (with labels and selectors) and define policies to access them
- E.g. in an application with a database, one service to group all frontend pods and one to group the database pod
- Services can expose Pods, ReplicaSets, Deployments, DaemonSets and StatefulSets
- Each service gets an in-cluster IP: `ClusterIP`
- `targetPort` for explicit mapping, otherwise the port defined in the Pod spec is used
- Pods each get own IP adresses and service handles forwarding traffic to them

### kube-proxy

- A node agent that watches the API server, it implements the Service configuration
- It configures iptables so that the API server and other cluster components can access other components
- Any node can receive external traffic and route it internally with `iptables`
- Each node has an `iptables` instance that stores complete routing roles for the cluster to ensure redundancy

### Tariff Policies

- Load-Balancing by `iptables` is random by default
- This does not guarantee that the selected Pod is the most efficient worker
- Two options: `Cluster` (target all ready endpoints of the service, default), `Local` (isolates only to the same node as the requester or the capturer of inbound external traffic)
- `Local` falls short, when no available ready endpoint is present and running on the Node

### Service Discovery

There are two methods to discover services:

- Env variables on the node (automatically added by `kubelet`)
- DNS add-on that allows for resolving domains like: `my-svc.my-namespace.svc.cluster.local`

Behaviour:

- e.g. frontend-svc:80 to access the frontend on port 80

### ServiceType

- `ServiceType` defines if pod is available from: within cluster-only, cluster and external, or maps to an entity inside or outside the cluster
- `ClusterIP` is the default type to allow access within the cluster
- `NodePort` can be used to make services available from outside by assigning a high Port on the Node
- `ExternalIP` where traffic is ingressed into the cluster through an outside IP (the routing has to be configured separately on each node, so that it can be reached by the outside IP)
- `ExternalName` returns a CNAME record of an externally configred service, used e.g. to make things like db.domain.de accessible to the cluster (even outside the namespace)

#### Deep-Dive: LoadBalancer

- `LoadBalancer` automatically creates NodePorts and ClusterIPs and routes to them
- Service is exposed at a static port on each worker node
- The underlying cloud providers load balancer feature is used (if there is none, this ServiceType falls back to NodePort behavior)

#### Deep-Dive: Multi-Port Services

- Allows e.g. to expose HTTP and HTTPS traffic
- If multiple ports are defined they have to be named `spec.port.name = $NAME`

Example:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: myapp
  type: NodePort
  ports:
    - name: http
      protocol: TCP
      port: 8080
      targetPort: 80
      nodePort: 31080
    - name: https
      protocol: TCP
      port: 8443
      targetPort: 443
      nodePort: 31443
```

#### Deep-Dive: Port Forwarding

- Allows for forwarding a local port to any application port (Service, Deployment, Pod)
- Used for testing mostly: `kubectl port-forward svc/frontend-svc 8080:80`

## Application Deployment

### Liveness Probes

- Applications may become unresponsive or delayed
- `kubelet` can utilize liveness and readiness probes to e.g. force restart or only forward traffic when live

Liveness Probes can be set by defining:

- Commands
- HTTP Requests
- TCP probes
- gRPC probes

#### Command

This spec defines such a probe command. A file is created regularly to check if the container can respond. initialDelaySeconds, failureThreshold and periodSeconds define intervals for the probe.

```yaml
spec:
  containers:
    - name: liveness
      image: k8s.gcr.io/busybox
      args:
        - /bin/sh
        - -c
        - touch /tmp/healthy; sleep 30; rm -rf /tmp/healthy; sleep 600
      livenessProbe:
        exec:
          command:
            - cat
            - /tmp/healthy
        initialDelaySeconds: 15
        failureThreshold: 1
        periodSeconds: 5
```

#### HTTP Request

This defines a request to `healthz` for HTTP request liveness

```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
    httpHeaders:
      - name: X-Custom-Header
        value: Awesome
  initialDelaySeconds: 15
  periodSeconds: 5
```

#### Other methods

- For TCP, a socket is opened, if that succeeds the application is considered healthy
- gRPC can be used if application implement the gRPC health checking protocol

### Readiness Probes

- In contrast to liveness probes, they define when applications can handle traffic
- Are defined similar to liveness probes but under `spec...readinessProbe`

### Startup probes

- For legacy applications that need a long time to start up
- Delay liveness and readiness probes

## Volume Management

- Some data is transient and destroyed with the Pod, for all other data there are volumes
- Volumes are storage abstractions masking different storage technologies
- Different container orchestrators had different methods of managing storage
- CSI is the now standardized interface for storage

The following volume types are supported by K8s:

- emptyDir (empty Volume on the worker node, is deleted when pod is destroyed)
- hostPath (shared dir with the host, stays up after pod destruction)
- gcePersistentDisk, awsElasticBlockStore, azureDisk, azureFile for the cloud providers
- cephfs, nfs for the corresponding file systems
- iscsi
- secret
- configMap (confiugration, shell commands, arguments)
- persistenVolumeClaim

### PersistentVolume subsystems

- PersistentVolumes provide APIs for users to consume persistent storage (which in turn is managed by admins)
- `PersistentVolumeClaim` is used to consume a resource type
- Volumes can be backed by several storage technologies (can be local to the pod, network, but also cloud or distributed storage)
- PersistentVolumes can be dynamically porivisioned based on a `StorageClass` (contains provisioniers, params to create a persistentVolume)

### PerisistentVolumeClaim

Is a request for storage by the user in four access modes:

- ReadWriteOnce (r/w by a single node)
- ReadOnlyMany (r by a many nodes)
- ReadWriteMany (r/w by a many nodes)
- ReadWriteOncePod (r/w by a single pod)

The volume can be mounted (depending on volume mode) as a directory or a raw block device. OOTB K8s does not support block storage, but that can be implemented with custom resource types.

### Process

1. Admin provisions persistent volumes pool
2. User requests a claim against the pool
3. User gets access to storage as soon as claim request is honored
4. The resource can be used in a pod (access depending on configuration)

## Configuration and Secrets

- Runtime parameters often need to be passed to containers, ConfigMap for configuration, Secrets for sensitive information

### ConfigMaps

Allow for decoupling configuration from the container image and can be created like: `kubectl create configmap my-config --from-literal=key1=value1 --from-literal=key2=value2`, which would lead to something like this in yaml (with key value pairs in `data`):

```yaml
apiVersion: v1
data:
  key1: value1
  key2: value2
kind: ConfigMap
metadata:
  creationTimestamp: 2024-03-02T07:21:55Z
  name: my-config
  namespace: default
  resourceVersion: "241345"
  selfLink: /api/v1/namespaces/default/configmaps/my-config
  uid: d35f0a3d-45d1-11e7-9e62-080027a46057
```

Creating it from a file is also possible, e.g.: `--from-file=values.properties`. They can then be used in Pods in different ways:

1. As env variables with `containers.envFrom.configMapRef.name` (full config map) or for specific variables
2. As volumes with `containers.volumeMounts.name=config-volume` (need to set mount path as well)

### Secrets

E.g. for DB passwords, API keys, .... `Secret` encodes the sensitive information in base64 before sharing. The secret is then referenced in other specs without exposing the content itself. Base64 is not encryption only encoding, so Secrets should not be committed to the source code.

Caution: Secret data is stored in plain text in `etcd`, so access to it must be limited or the secret data must be encrypted at rest (must be enabled at the API server level).

We can create Secrets imperatively: `kubectl create secret generic my-password --from-literal=password=mysqlpassword` and show infor about them with: `kubectl describe secret my-password`.

They can also be set declaratively (secret values has to be base64 encoded, e.g. `echo password123 | base64`):

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-password
type: Opaque
data:
  password: $BASE64_STRING
```

You do not need to base64 encode if you use `stringData`:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-password
type: Opaque
stringData:
  password: mysqlpassword
```

Secrets can also be created from a file: `kubectl create secret generic my-file-password --from-file=password.txt` (password.txt just has the single password as content).

Similar to ConfigMaps, secrets can be used with `valueFrom.secretKeyRef` (with `secretKeyRef.name` as the secret name and `secretKeyRef.key` as the specific key inside the referenced secret) in `containers.env`. The secrets can also be mounted as volumes.

## Ingress

Another layer of API abstraction deployed in front of services. Represents a unified method of managing access to applications from the external world.

- Decouples routing rules from applications to allow for updating applications without concern about its external access
- Ingress configures a Layer 7 HTTP/S load balancer providing: TLS, name based virt hosts, loadbalancing, fanout routing, custom rules
- With Ingress users do not connect directly to a service, but Ingress decides how requests are forwarded

Name based virtual hosting can be defined as follows:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    nginx.ingress.kubernetes.io/service-upstream: "true"
  name: virtual-host-ingress
  namespace: default
spec:
  ingressClassName: nginx
  rules:
    - host: blue.example.com
      http:
        paths:
          - backend:
              service:
                name: webserver-blue-svc
                port:
                  number: 80
            path: /
            pathType: ImplementationSpecific
    - host: green.example.com
      http:
        paths:
          - backend:
              service:
                name: webserver-green-svc
                port:
                  number: 80
            path: /
            pathType: ImplementationSpecific
```

There is also the option of using Fanout rules to use one Ingress managed load balancer that then fans out to different services:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    nginx.ingress.kubernetes.io/service-upstream: "true"
  name: fan-out-ingress
  namespace: default
spec:
  ingressClassName: nginx
  rules:
    - host: example.com
      http:
        paths:
          - path: /blue
            backend:
              service:
                name: webserver-blue-svc
                port:
                  number: 80
            pathType: ImplementationSpecific
          - path: /green
            backend:
              service:
                name: webserver-green-svc
                port:
                  number: 80
            pathType: ImplementationSpecific
```

### Ingress Controller

- An application watching the Control Plane Node's API server for changes in Ingress resources
- Also called Ingress Proxy, Service Proxy, Reverse Proxy
- There are a few different supported controllers: GCE, AWS, Nginx (deprecated), Traefik, ...
- The actual supported implementation is defined by `spec.ingressClassName`

## Advanced Topics

### Annotations

- Allow for attaching arbitrary metadata to any objects in a K/V format
- Annotations are not used to identify and select objects (like labels) but for: Build/Release IDs, PR numbers, git branch, author metadata, pointers for logging, monitoring, deployment state, ...

### Quota and Limits management

`ResourceQuota` API resource can be used to fairly divide resources to users. These are the types it supports:

- Compute Resource Quota (total sum of CPU, memory, ... resources for a given namespace)
- Storage Resource Quota (limit sum of storage resources)
- Object Count Quota (limit number of Pods, ConfigMaps, PersistentVolumeClaims, ...)

A spec file could look like this:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-resources
  namespace: myspace
spec:
  hard:
    requests.cpu: "1"
    requests.memory: 1Gi
    limits.cpu: "2"
    limits.memory: 2Gi
    requests.nvidia.com/gpu: 4:
```

There is also a `LimitRange` resource to set ranges for resource limits (e.g. between 500m and 2000m).

In addition to these limits, there are more complex, dynamic scaling solutions:

- HPA (Horizontal Pod Autoscaler) to define rules like: Scale from 2 to 10 replicas if one replica reaches 80% utilization (`kubectl autoscale deploy myapp --min=2 --max=10 --cpu-percent=80`)
- VPA (Vertical Pod Autoscaler) automatically sets resource requirements based on historical data, current availability and real-time-events
- Cluster Autoscaler can resize the cluster as a whole to allow for more but also less resources

### Jobs and CronJobs

- A Job creates one more pods to perform a given task
- It makes sure that a given task is completed successfully (responsible for Pod failures)
- If a task is complete all pods for this job are terminated automatically
- Some config options are: `parallelism`, `completions`, `activeDeadlineSeconds`, `backoffLimit`, `ttlSecondsAfterFinished`
- From 1.4 scheduled Jobs can be run with CronJobs

### StatefulSets

- E.g. for MySQL cluster
- Very strict on service and storage dependencies
- Supports scaling, rolling updates, rollbacks

### Custom Resources

- Custom resources are dynamic in nature, and they can appear and disappear in an already running cluster at any time
- K8s source does not have to be modified when using custom resources
- Custom Controller is required for a custom resource (interprets resource structure and performs required actions)
- Two ways of adding: Custom Resource Definitions (programming only for controller), API Aggregation (sit behind API server and programmatically add on requests to the API server)

### Security Contexts

Can be applied by using `Pod Secrity Admission`. To define specific privileges and access control settings (e.g. host user, ...) like this:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: demo
spec:
  securityContext:
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
  volumes:
    - name: vol
      emptyDir: {}
  containers:
    - name: busy
      image: busybox:1.28
      command: ["sh", "-c", "sleep infinity"]
      volumeMounts:
        - name: vol
          mountPath: /data/demo
      securityContext:
        allowPrivilegeEscalation: false
```

### Network Policies

- Pods communicate freely without restrictions in a namespace
- Network Policies can limit how the pods can communicate similar to firewalls
- Can limit incoming and outgoing traffic with `ipBlocks` and `ports`

This is a sample NetworkPolicy, that allows inbound traffic to 6379 in the namespace `project=myproject` from pods labeled `role=frontend`. Outbound traffic is allowed from the protected ports to TCP port 5978 in the IP range `10.0.0.0/24`:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: demo-netpol
  namespace: default
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              project: myproject
        - podSelector:
            matchLabels:
              role: frontend
      ports:
        - protocol: TCP
          port: 6379
  egress:
    - to:
        - ipBlock:
            cidr: 10.0.0.0/24
      ports:
        - protocol: TCP
          port: 5978
```

### Monitoring, Logging and Troubleshooting

There are two popular monitoring solutions:

- Kubernetes Metrics Server (cluster-wide aggregator of resource usage data) `kubectl top nodes/pods` can be used to get logs
- Prometheus (can also be used to scrape resource usage and be handled by its client libraries)

For Logging K8s does **not** provide cluster-wide logging. Third party tools need to be used to collect logs, e.g. `Elasticsearch`/`Fluentd`/...
Normally only running container logs can be accessed with `kubectl logs pod-name container-name`, so 3rd party tools are needed.

### Helm

- To deploy complex applications a large number of manifests is needed
- Deploying them one by one can be hard
- Helm Charts can be used to templatize manifests in a well-defined format
- Charts can be served via repositories
- Helm is both a package manager for K8s, as well as a CLI client
- The helm client queries chart repositories base on search params and then requests the API server to deploy clusters

### Service Mesh

- Third party solution to Kubernetes native application connectivity
- Features include service discovery, mutual TLS, multi-cloud routing, traffic telemetry
- Relies on proxy of the Data Plane which is managed by the Control Plane
- Control Plane runs agents for the features descibed above
- Data Plane proxy is typically injected into pods and handles pod-to-pod communication as well as communication to the Control Plane of the Mesh
- Some implementations are: `Consul`, `Istio`, `Kuma`, `Linkerd`, `Traefik Mesh`

### Application Deployment Strategies

More complex deployment mechanisms than rolling update/rollback might be necessary, such as:

- Canary
  - Two applications run simultaneously managed by two deployment controllers, exposed by the same service
  - Users can manage the amount of traffic each deployment is exposed to by scaling up or down deployment controllers
  - Resulting in more/less replicas for one of the deployments)
- Blue/Green
  - Two applications in two environments, where one is active one is idle (or currently tested)
  - Two independent deployment controllers with two Services, one each
  - Typically Ingress or another service is used to direct traffic to one deployment
