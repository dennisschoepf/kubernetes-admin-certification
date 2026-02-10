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
