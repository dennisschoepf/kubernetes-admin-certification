# Introduction to Cloud Infrastructure Technologies (LFS151)

## Deployment Models

- Private: Only for one organization, hosted/managed externally or internally, e.g. with OpenStack
- Public: Open for anyone, AWS/Azure/GCP/...
- Hybrid: e.g. storage of sensitive information in private, public services on public cloud

**Other models**:

- Distributed: Multiple systems that form a single network
- Multicloud: Multiple providers to e.g. avoid vendor lock in
- Polycloud: Like multicloud but rather used as description when different services from each provider are used

## Virtualization

**Types:**

- Type-1 Hypervisor (Native/Bare-Metal): Runs directly on hardware without the need for a Host OS (e.g. AWS Nitro, IBM z/VM, ...)
- Type-2 Hypervisor (Hosted): Runs on the host OS (e.g. VMWare, VirtualBox, ...)
- Mixed: e.g. KVM that runs as a kernel module
- Emulators: e.g. QEMU that emulates a system in userland to e.g. run on unsupported hardware

**Characteristics:**

- Allow for sharing hardware (CPU, disk, network, RAM) -> Hardware virtualiziation for CPUs is supported by most modern CPUs, sometimes even nested virtualiziation
- Allow for different OS installations on VMs

### KVM

- Kernel module that can be loaded to turn the kernel into a hypervisor capable of managing guest machines
- Hardware virtualiziation extensions have to be available and need to be enabled (AMD-), available for x86, but ported to FreeBSD, ARM and more
- Process: Linux Kernel manages hardware access, KVM module manages hardware access of KVM guests and things like I/O requests (with `iothread`)
- **Caution:** KVM handles abstraction of network and disk but **not** CPU, this is done by userland tools like QEMU or `virt-manager` that access the `/dev/kvm` interface
- Allows for "overcommitting" = Allocate more resources than are available at the host (through dynamically allocating unused resources from other guests)

**Benefits:**

- Highly scalable OSS solution that is "safe" (SELinux)
- Allows for efficient "hardware-assisted" virtualiziation

### VirtualBox

- Type-2 Hypervisor for select guest OSes, can be compiled and inserted as a module into the linux kernel
- OSS userland solution that is capable of both software-based virt as well as hardware-assisted virt

### Vagrant

- VMs (images downloadable as `boxes`) for dev environments that can be shared within a team and declared with a `Vagrantfile`
- Allows for management of multiple projects in an isolated and restricted environment
- Accepts `providers` that run the VMs (e.g. VirtualBox, Hyper-V, VMWare, or even Docker)
- Similar to docker allows for synced folders, provisiniong steps, plugins, networking config

## IaaS

- Cloud service model that provides on-demand physical and virtual computing resources (storage, netowrk, firewall, load balancers)
- To provide virtual resources hypervisors like KVM, Hyper-V, ... are used

### AWS EC2

- EC2 instances are Virtual Machines running on type-1 hypervisors, but also KVM and Nitro
- AMIs are preconfigured images with packages installed
- Different instance types optimized for: Compute (CPU), Accelerated Compute (GPU), Memory (RAM), ...
- VPC for network isolation, security groups to manage inbound/outbound traffic, EBS for persistent storage attachment
- CloudWatch for monitoring
- Elastic IP to map to a static IP

**Benefits:**

- Pay only for the time and resources used
- Integrates well with other AWS components
- Specialized instances

### Azure

- Similar to AWS in all aspects

### DigitalOcean

- Droplets are Linux based VMs on top of a KVM type-1 hypervisor with SSDs
- One-click installation of application stacks (e.g. Kubernetes, Jenkins, ...)

### GCP

- GCE (compute engine) instances are virtual machines running on a KVM type-1 hypervisor
- Very similar to AWS and Azure

### OpenStack

- Self-Hostable cloud-computing platform
- Create instances/VMs on demand
- Can be used to create public and private clouds

## PaaS

- Level above IaaS - Allows for running applications without being concerned by the infrastructure
- Sample providers: Cloud Foundry, OpenShift, Heroku (discontinued)
- Typically allow for application portability, auto-scaling, isolation, centralized logging, dynamic routing, application health management, horizontal & vertical scaling
- Directly deploy a number of supported application stacks: e.g. Java, Node.js, PHP, Go with a simple command (e.g. `cf push` for CloudFoundry)
- CloudFoundry built on top of other providers and/or Kubernetes (`korifi`), OpenShift based on Kubernetes

## Containers

- Solve the problem of un-isolated applications on a host (VM or bare-metal)
- Allow for running multiple isolated instances in user-space
- Include application source code, required libraries and runtime as an `image` (does not contain kernel-space components)

### Building Blocks

**Namespaces:**

- Global system resources are wrapped into an abstraction, so that it appears to them that they have their own isolated instance of a global resource
- `pid`: Each namespace has its own PID 1, each namespace have the same PIDs
- `net`: Each namespace has its own IP address and network stack
- `mnt`: Each namespace has its own view of the filesystem
- `ipc`: Each namespace has its own interprocess communication
- `uts`: Each namespace has its own hostname and domain name
- `user`: Each namespace has its own user and group ID number spaces, root within the container is not the hosts root

**cgroups:**

- Control groups to organize distributed system resources, for linux: `blkio`, `cpu`, `cpuacct`, `cpuset`, `devices`, `freezer`, `memory`

**Union Filesystem:**

- Allows `layers` to be transparently laid upon each other to create a new virtual filesystem
- At runtime a container is made of multiple layers to create a read-only filesystem
- Container also gets an ephemeral read-write layer local to the container

### Container Runtimes

- `runC`: CLI Tool to spawn and run containers according to OCI specs
- `crun`: faster `runc` written in C
- `containerd`: The one that docker uses - Can run a daemon and manage the entire lifecycle of containers
- `CRI-O`: OCI-compatible implementation of the K8s CRI (high-level alternative to Docker for Kubernetes)

### Containers vs. VMs

- VMs share operate on a layer upon bare-metal or the host OS and host a complete guest OS
- Containers are isolated but share the OS and bins/libraries on the OS
  - Containers have to be compatible with the host OS as there is no hypervisor layer between
  - Pro: light footprint, can be deployed fast, scalable, use less memory than a VM, portability

### Docker

- Docker Platform is a collection of dev tools that follow a client-server architecture with a docker client connecting to a docker host server
- The docker host server runs the docker daemon and executes client requests (like adding, removing, starting, stopping, ...)

### Podman

- Open-Source, **daemonless** tool that works with OCI containers, relies on an OCI compliant runtime like `runc`
- In contrast to docker it does not need a daemon like `containerd`, it can run `runc` directly
- Runs rootless containers by default
- Uses Containerfile (very similar to Dockerfile)
- Uses two additional tools: `buildah` for container image builds and `skopeo` for working with images locally and on remote

### Micro OSes for containers

- Eliminate system packages that are not essential for running containers and applications within containers

#### Alpine

- non-commercial distribution
- `apk` package manager
- 5-8 MB per container, can be run in `diskless` (entire system in RAM), `data` (everything but `/var` in RAM) and `sys` ("normal") mode
- different flavors: `standard`, `extended`, `netboot`, `virtual`, ...

#### BusyBox

- between 1 and 5MB per container often for embedded systems, not an OS in itself

#### Fedora CoreOS

- minimal OS for running containerized workloads, updates automatically
- is runnable in clusters and standalone instances -> optimized for Kubernetes
- does not have a package manager
- Multiple installation methods: Cloud-launchable (e.g. directly through AWS)
- Includes:
  - `ignition` to manipulate disk during early boot (partitioning, writing files, configuring users) in `initramfs` (runs before userpace boot to allow advanced admin features)
  - `rpm-ostree` to manage immutable filesystems (two, one for boot and one for fetching updates)
  - SELinux hardening

#### Flatcar

- Container-optimized OS with immutable filesystem and automated atomic updates
- Drop-In replacement for Red Hat CoreOS Container Linux
- Works with `ignition`
- Supports `containerd` and other runtimes for clusters

#### k3OS

- Distribution aiming to minimize OS maintenance tasks in a Kubernetes cluster
- Designed to work with k3s
- Fast installation, fast boot, does not require package manager and can be managed through Kubernetes
- Optimized to run in a low-resource computing environment
- can be configured with `cloud-init`

#### Ubuntu Core

- Mainly for embedded but also found in container deployments, around 260MB
- Immutable packages, persistent digital signatures, isolated applications, read-only FS
- Works wit snaps
- Strict application isolation

## Container Orchestration

- Allows for containerized workloads to be managed with high availability and fault tolerance
- Needed functionality includes: Grouping multiple hosts together, schedule containers to run on specific hosts, containers communicating between each other, dependent storage for containers, access containers over names instead of IPs

### Kubernetes

- Allows for automated deployment, operations and scaling of containerized applications
- Supports different container runtimes: `containerd`, `cri-o`, `docker engine`, `mirantis`

#### Components

- Cluster: a collection of bare-metal or virtual systems and other infrastructure resources
- Control-Plane Node: System that does scheduling decisions, manages worker nodes, enforces ACPs and reconciles changes, includes: `kube-apiserver`, `etcd`, `kube-scheduler`, `kube-controller-manager` (can be multiple nodes for hight availability)
- Worker Node: System where containers are scheduled to run in `Pods`, runs a daemon called `kubelet` that is controlled by `kube-apiserver`, `kube-proxy` handles networking (e.g. applications accessible from external requests, internal routing)
- Namespace: Partition the cluster into virtual-subclusters by segregating resources (for multi-tenancy)

#### API Resources

- Pod: Workload Management Unit, can handle a group of containers with shared dependencies (e.g. storage), mostly used for single-container-management with its Secrets and ConfigMaps (! does not handle self-healing, scaling, ...)
- ReplicaSet: Operator that manages lifecycle of pods, rolls out pod replicas, ensures desired number of pods are running, self-heals if pods are lost
- Deployment: Top-Level-Controller allowing declarative updates for Pods and ReplicaSets (e.g. number of replicas, update mechanism, roll back, scale, pause, resume deployments)
- DaemonSet: Controller that manages the lifecycle of node agent pods, rolls out desired amount of pod replicas while ensuring that each cluster node will run exactly one application pod replica
- Service: Traffic routing unit (implemented with `kube-proxy`) that allows for load-balancing, DNS name registrations for applications, name resolution to private/cluster internal static IP. Can reference pods, replicaSets, Deployments, DaemonSets
- Label: Arbitrary KV-pair attached to resource - allow for logical grouping (e.g. `app`-label for apps)
- Selector: Allow controllers/operators to search for groups of resources described by labels
- Volume: Abstraction Layer for container storage management, e.g. used to mount local host storage, network storage, distributed storage clusters or cloud storage services

#### Hosted Solutions

- Amazon EKS
- Azure Kubernetes Service (AKS)
- Google Kubernetes Engine (GKE)
- RedHat OpenShift
- IBM, NetApp, Vultr, VMWare
- There are also managed kubernetes solutions, e.g. by Canonical, Rancher, ...

##### Amazon EKS

- Allows for cluster autoscaling (dynamically adding worker nodes)
- Integrates with Kubernetes Role-Based Access Control (RBAC) and AWS IAM authentication
- Users do not manage the Kubernetes control plane

### Docker Swarm

- Groups multiple docker engines into a cluster (=swarm)
- Consists of swarm manager nodes (accept commands and schedule workloads) and swarm worker nodes (run a docker engine to run the container workload)
- Supports docker networking and volumes, failover and rolling updates
- Uses a declarative approach for defining services of the application stack, swarm manager nodes reconcile differences between actual state and desired state
- TLS communication between nodes
- "Swarm" is discontinued but Docker Swarm Mode is still available and working

### Nomad

- Cluster manager and resource scheduler from HashiCorp for running microservices and batch jobs
- Can handle docker containers, VMs, unikernels and Java applications
- Multi-Datacenter and multi-region support, rolling upgrades, can run together with Kubernetes, integrates with other HashiCorp solutions
- Blue-green and canary deployments supported by job files

## Amazon ECS

- Three launch modes that define the desired infrastructure
- **Fargate**: Runs containers without managing servers and clusters -> just package containers with CPU, memory, networking and IAM policies
- **EC2**: manages the cluster itself (provisioning, patching, scaling the underlying cluster)
- **External**: Run containerized applications on-premises (allows for e.g. hybrid cloud, where public cloud is ECS and private cloud is ECS Anywhere hosted on own servers)

## Unikernels

- As part of the containers' running process it is necessary to run user-space libraries of the distribution, a unikernel allows for a very minimal image where even at the kernel level only strictly necessary functionality is included
- They are single-address-space machine images constructed by using library operating systems and can be deployed as a VM
- Typically only include: application code, configuration files, user-space libraries needed for the application, application runtime (e.g. JVM), system libraries to allow for communication with the hypervisor
- According to the protection ring of the x86 architecture, we run the kernel on ring0 and the application on ring3, which has the least privileges. ring0 has the most privileges, like access to hardware, and a typical OS kernel runs on that. With unikernels, a combined binary of the application and the kernel runs on ring0
- Unikernel images typically run directly on a hypervisor or bare metal
- Allows for faster boot time, more applications per host, efficient resource utilization, simple development and management model, reduced attack surface

## Microservices

- Already did a lot of research on this

## Software-defined Networking (SDN)

- Growing number of devices, applications, VMs, containers lead to greater need in flexible, virtualized networking, just networking per host is not enough
- SDN decouples the network control layer from the traffic forwarding layer to create custom networking rules
- There are three planes: `Data` (or Forwarding) -> handles data packets and applies actions based on rules, `Control` -> calculating and programming actions for Data Plane, forwarding is decided here and also QoS and VLANs are implemented here, `Management` -> where we can configure, monitor and manage network devices
- These three planes allow a network to serve three distinct activities: `Ingress and egress packets` at the lowest level (Data Plane with switches, modems), `Collect, process, manage network information` at the Control Plane, `Monitor and manage the network` on the Management Plane
- Process: Management Plane calls (and is called) by the Control Plane which handles Network Services, the Control Plane then submits lookup tables to the Data Plane which handles the actual network devices

## Networking for Containers

- A (linux) host uses ([Network Namespace](https://lwn.net/Articles/580893/)) kernel feature to isolate networks from multiple containers on a system
- Network namespaces can be shared netween containers (extensively used by Kubernetes)
- On a single host `veth` (virtual ethernet) can be used to give a virtual network interface to each container and assign IP addresses
- Multi-Host-Networking can be achieved with an overlay network driver that encapsulates Layer 2 traffic (from MAC address to MAC address) to a higher layer (Layer 3 = Routing based on IP addresses), examples are `Docker Overlay Driver`, `Flannel`, `Weave`

### Container Networking Standards

- `CNM`: Docker-initiated networking model using `libnetwork` that has different modes: `Bridge`, `Swarm/Overlay`, `Remote`
- `CNI`: CNCF project that consists of speccs and libraries to write plugins for Linux container network interfaces. It is limited to providing network connectivity of containers and removing allocated resources when the container is deleted. Used by Kubernetes

### Service Discovery

Is a mechanism that enables processes and services to find eacht other automatically. In container-land it is used to map the **container name** to an **IP address**, so that it can be accessed by name. Discovery is generally implemented in two steps:

1. Registration: When the container starts, the scheduler registers the container name <-> IP mapping in a key-value store like etcd ord Consul. Mappings are updated on restarts and stops
2. Lookup: Services and applications use Lookup to retrieve the IP address (generally supported by DNS local to the environment -> looks up values in the key-value store)

## Docker Networking

### Single Host Networking

- There are distinct network drives on a single docker host: Bridge, Host, Null
- Network Namespaces can be shared among containers (by starting subsequent containers with `docker container run --net=container:container_name`)

#### Bridge

A software bridge network on a host. It forwards traffic between two networks based on hardware addresses. Docker creates a `docker0` network bridge by default. Every container receives a unique IP from this bridge network (unless specified otherwise). Internally Docker uses `vEth` to create two virtual interfaces (attached to both the container network interface and the bridge network interface).

Can be inspected within the container with `ifconfig | grep "docker0"` (or for the network as a whole `docker network inspect bridge`). Custom bridges can be created with `docker network create --driver bridge bridge_name` and containers attached with `docker container run --net=bridge_name -itd --name=container_name alpine-slim`. Already running containers can be connected to a network with `docker network connect`.

#### Null

The container is not attached to any networks, it just has a loopback interface and is not accessible from other networks.

#### Host

Allows for sharing the host network namespace with a container. NOT RECOMMENDED as the containers would have full access to the host network

### Multi Host Networking

When containers are part of a swarm Docker allows cross-host communication between containers with two different drivers:

- Docker Overlay Drivers: Docker encapsulates a container IP packet in a host packet, on another host docker decapsulates the packet and forwards it to the receiving container (built with libnetwork)
- Macvlan: A MAC address is assigned to each container, so that it appears as a physical device on the network -> A IP from the network subnet can be assigned to each container. Direct container-to-container communication is possible with this. Only available if the hardware supports it

## Podman Networking

Supports different drivers for container networks: Bridge in rootless mode and bridge, macvlan, and ipvlan in rooted modes (because macvlan and ipvlan need access to the host network). Operations:

- `podman network create --subnet 192.168.20.0/24 --gateway 192.168.20.3 bridgenet`
- `podman netwrok inspect bridgenet`
- For macvlan `podman network create -d macvlan -o parent=eth0 macvnet`

## Kubernetes Networking

- Kubernetes assigns a unique IP address to each Pod
- Containers in a Pod share the same network namespace and can refer to each other by localhost
- Containers in a Pod can expose unique ports
- Kubernetes assumes that Pods can communicate with each other through their IP address irrespective of the nodes they are scheduled on (using CNI wihtout NAT)
- CNI defines that all pods on a node can communicate with all pods on all nodes without NAT, agents on a node can communicate with all pods on that node & pods in the host network can communicate with all pods on all nodes without NAT

Projects implementing CNI are SDN implementations. There are several different networking plugins available:

- AWS VPC CNI allows for private cloud networking for cluster networks (VPC flow logs, routing policies, and security groups)
- Calico uses BGP for Kubernetes networking
- Cilium for secure connectivity (can enforce network policies on L3-L7)
- Flannel or OVN Kubernetes (which uses Open VSwitcdh for network policy and load balancing) using an overlay network
- Weave net as a simple network without additional configuration

Multi-Host networking on docker allows for:

- Communication between pods with virtual (and ephemeral) pod ClusterIP
- Communication between pods with ServiceClusterIP (can change)
- Communcation between pods through the Cluster NDS server by using the pod name

## Storage Management for Containers

- SDS (software-defined-storage) is storage virtualization built to separate storage hardware from the software that accesses it
- Physical hardware can be combined and managed as a single storage pool
- SDS may provide resiliency features like replication, erasure and snapshots
- Within a storage cluster multiple access methods are possbile (File, Block, Object)

Different Implementations are available.

### Ceph

- Everything stored as objects
- Uses CRUSH algorithm to find, write and read
- RADOS as object store (makes sure that everything is in a consistent state, handles replication, failure detection, recovery, data migration, ...)
- Different interfaces on top of RADOS (Block Device, POSIX compliant filesystem, ...)
- Open Source, runs on any commodity hardware
- High throughput by stripping files/data across multiple nodes

### Gluster

- Machines are grouped in a trusted pool, directories from those machines in a GlusterFS volumes using FUSE
- Different types are possible: Distributed, Replicated, Distributed & Replicated, Dispersed, Distributed & Dispersed
- Does not have a centralized metadata server
- Can be accessed with native fuse mount as a NFS, or CIFS
- Scales to several petabytes and supports Object, Block and Filesystem storage

### Storage Management in Docker

- We need to make sure the data volume required by the container is available on the host on which the container is scheduled to run
- Docker uses `copy-on-write` when starting containers from images, the image is on a read-only filesystem layer
- Changes performed by the container are on a writable filesystem layer
- Docker Images & Containers are stored on the host system
- Different drivers are available for storage: Btrfs, Device Mapper, Fuse-Overlay (for rootless), Overlay2, VFS, ZFS

There are different options for storing files on the host:

- Volumes: Are stored in `/var/lib/docker/volumes` and directly managed by docker `docker container run ... -v webvol:/webdata` uses the volume at `/var/lib/docker/volumes/webvol/_data` and mounts it as `/webdata`
- Bind Mounts: Allows docker to mount any file/directory from the host into the container with `docker container run ... -v /mnt/webvol:/webdata` uses the hosts `/mnt/webvol` directory and mounts it as `/webdata`
- Tmpfs: Stored only in host memory (for non-persistent state data)

In addition there are different docker volume plugins to mount e.g. Block Storage Providers, GlusterFS or others.

### Storage Management in Podman

- Uses volumes similar to docker
- Supports different storage drivers like: AUFS, Btrfs, Thinpool, Overlay, VFS, ZFS

Data can be managed in a variety of ways:

- Volumes (`/var/lib/containers/storage/volumes`)
- Bind (named volume from the host system)
- Tmpfs (similar to docker)
- Image (to mount an OS image file)
- Devpts (to mount a FS storing telnet, ssh, xterm data)

### Volume Management in Kubernetes

- Kubernetes uses volumes to attach external storage to containers managed by pods
- Volume is a directory backed by a storage medium where its contents are determined by the volume type
- Volume is linked to a Pod and shared among containers of that Pod
- Volume has the same lifetime as the Pod, but outlives the containers of that Pod
- If the pod is deleted the volume and all its data is lost

There are different volume types that decide the size and content type of the volume. As of 2023 Kubernetes moves to CSI (Container Storage Interface), which means that Kubernetes operators must download an install the desired CSI drivers when needed (cephfs, azureFile, awsElasticBlockStore, nfs, secret, ...).

Kubernetes groups storage with the persistent volume subsystem, which provides APIs to manage/consume the storage. The `PersistentVolume` resource type is used for managing the volume, while `PersistentVolumeClaim` is used to consume the volume. Persistend volumes can be provisioned statically or dynamically.

For dynamic provisioning of these volumes the `StorageClass` resource is used, whcih contains predefined provisioners and parameters.

#### Process: PersistentVolumeClaim

1. Administrator provisions persistent volumes
2. User Requests a `PersistentVolumeClaim`
3. Claim Request is honored
4. PVS is mounted into a Pod and accessible by containers
5. If the tasks are completed and the pod is deleted, the PVC may be detached from the PV to release it for future usage

#### Adding Storage Volumes to Pods

You can attach an ephemeral volume in the Pod definition:

```
apiVersion: v1
kind: Pod
spec:
  containers:
  - command:
    - bash
    - -c
    - sleep 3600
    image: ubuntu
    name: storage-pod
    resources: {}
    volumeMounts:
    - name: vol
      mountPath: /data
  volumes:
    - name: vol
      hostPath:
        path: /tmp/data-vol
        type: DirectoryOrCreate
  nodeName: ...
```

`nodeName` ensures that the Pod is scheduled on a specific node where the volume path will be created on the host when deploying the Pod.

#### Distributed Storage Management

When scaling up dramatically, manual PersistentVolume configuration is tedious and the cluster cannot be moved easily. There are projects that aim to abstract away the storage infrastructe from the storage access. Open source options are:

- Rook (using ceph or others it allows for deploying, scaling, self-healing, ... storage solutions)
- Longhorn (highly available persistent storage for K8s with backups, snapshots, cross-cluster recovery)

### Container Storage Interface

Mentioned before this is a standard for storage volumes. It defines APIs for provisioning, attaching, mounting, consuming and snapshotting of storage volumes so that different orchestrators (K8s, Nomad, ...) are able to easily connect to different external storage volumes by other vendors.

## DevOps and CI/CD

The practice of configuring and enabling of various development workflows such as: Continuous Integration, Continuous Delivery, Continuous Deployment with popular options being:

- Jenkins (OSS, supports CI & CD, a lot of plugins)
- Argo
- GitLab

### Cloud Native CI/CD

- Meaning the CI/CD pipeline IS code and stored along or besides the source code.
- DevOps (automating CI/CD) vs GitOps (Git as source for operations configuration and code) vs DevSecOps (the practice of integrating security into all CI/CD processes)
- Tools include: Helm, Argo, GitLab, Flux, CircleCi, Jenkins X

## Configuration Management
