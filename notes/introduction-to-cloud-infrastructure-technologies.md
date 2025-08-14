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
