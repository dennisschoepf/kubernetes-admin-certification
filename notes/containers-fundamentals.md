# Container Fundamentals

## Virtualization

### Control Groups

- Kernel feature that allow the limitation, accounting and isolation or resources used by groups of processes
- Mostly used to manage multiple procceses together
- They can set usage quotas and prioritize groups to receive more CPI time or memory
- They can also be measured, frozen or restarted
- cgroups can be listed with `sudo lscgroup` (`cgroup-tools` needed)

`freezer`, a system cgroup that allows a group of tasks to be suspended and resumed, can be interacted with like this:

```sh
cd /sys/fs/cgroup/freezer/
sudo mkdir mycgroup
cd mycgroup
ls
```

The tasks file holds PIDs of processes associated with the `cgroup` (`cat tasks` inside the dir of `mycgroup`). Processes can be managed by adding the process ID to the `tasks` file `sudo bash -c 'echo $PID >> tasks'`.

By writing to a file of the cgroup processes in the `tasks` file can be frozen:

```sh
sudo bash -c 'echo FROZEN > freezer.state' # applied to all PIDs in the tasks file
sudo bash -c 'echo THAWED > freezer.state' # Thawed resumes the processes
```

### Namespaces

- Kernel feature allowing groups of processes to have limited visibility of host resources
- May limit cgroups, hostname, process ID, IPC, network interface, routes, users, fs visibility
- In container context
  - Namespaces isolate processes from different containers
  - Prevent containers from modifying resources used by other containers
  - Processes isolated in a container can only see system resources for that container
  - PID conflicts are resolved by running processes in different namesspaces (globally they get different PIDs)
  - Allow for e.g. multiple root processes on a host, which is not possible without namespaces

Namespaces can be listed as follows:

```sh
sudo ls -l /proc/<PID>/ns/
```

And e.g. network namespaces created like this:

```sh
sudo ip netns add namespace1
sudo ip netns add namespace2
ip netns list
sudo ip link add veth1 type veth peer name veth2 # creates two interconneccted virtual ethernet devices
sudo ip link set veth1 netns namespace1 # Links to the namespace
sudo ip netns exec namespace1 ip link set dev veth1 up # Brings the devices up
sudo ip netns exec namespace1 ip addr add 192.168.1.1/24 dev veth1 # assigns IPs to the devices

sudo ip netns exec namespace1 ping -c5 192.168.1.2 # Should resolve if veth2 and namespace to is set accordingly

sudo ip netns delete namespace1 # cleans up the namespace
```

### UnionFS

- Allows the overlay of separate transparent file systems to produce a single unified file system
- Several file systems (called branches) are virtually stacked and their contents appear to be merged
- Physically the contents are separate which (together with Copy-On-Write and read-only and read-write access modes) helps to prevent data corruption

The CoW process is as follows:

1. CoW create a physical copy of a file when it should be written to
2. A copy of the file is made onto a writable file system
3. Modifications access the copy
4. Through the unified file system it appears as if the original file has been modified

In containers, unionfs allows for changes to be made to the container image at runtime. unionfs gives the impression that the image is modified (saved to a writable system), while leaving the base image intact.

UnionFS can be interacted with as follows (`unionfs-fuse` is needed):

```sh
mkdir dir1 && touch dir1/f1 && touch dir1/f2 && mkdir dir2 && touch dir2/f3 && touch dir2/f4
mkdir union # mount point for unionfs
unionfs dir1/:dir2/ union/ # overlays dir1 and dir2 in union/
ls union/ # lists all files: f1 f2 f3 f4
```

### Full Virtualization vs OS virtualization

VMs are created on top of hypervisors (either bare-metal or on a host) that emulate hardware:

- Extensive overhead to reach physical hardware or the outside world
- Many layers of abstraction (guest OS, hypervisor, host OS)
- Allows for many VMs in parallel (depending on hypervisor)

Containers:

- Only virtualize and isolate resources for an application
- Application is boxed and shipped only with its dependencies by creating a container image
- Once deployed the container directly runs on the host without a guest OS and hypervisor
- The host OS must provide isolation and resource allocation via its OS-level virt capabilities, so the user space component of the container must be compatible with the host OS

### Operating System-Level Virtualization

- Refers to a kernel's capability to allow for creation and existence of multiple isolated virtual environments
- These environments encapsulate running programs and conceal the nature of their enviroment (creating the illusion of a real compouting env)
- Where a real OS may allow or deny resource access, OS level virt may limit programs access to host OS resources
- Typically used to limit usage and securely isolate resources shared between multiple users/programs and to separate programs two run in their own virtual environments
- OS-level virt requires less overhead, but the virtual enviroments are limited to the host OS

There different mechanisms of achieving OS-level virt:

- chroot
  - changing apparent root directory of a process (chrooted directory)
  - any process running inside chrooted dir is under the impression that it is running as root
  - process cannot escape to the real root directory of the operating system
  - can be used for test environments, isolated dependencies or to recover unbootable systems
  - only root user may perform chroot, but it does not guard against root attacks
  - FreeBSD jails can be used to mitigate that (isolate FS, processes and users)
  - chroot does not isolate users, i/o, network, hostname, ...
- OpenVZ
  - only available on linux
  - allows a host to run isolated virtual instances (=containers)
  - OpenVZ containers behave like physical hosts (more of a VM/container hybrid)
  - access to real hardware must be enabled explicitly
- Linux Containers (LXC)
  - allow multiplate isolated systems running on one host using cgroups and chroot together with namescpaces
- systemd-nspawn
  - can be used to run a simple script or boot an entire Linux-like OS in a container
  - containers are fully isolated from each other and the host so containers cannot communicate directly with each other

#### chroot

Can be used as follows (with `debootstrap`):

```sh
sudo mkdir /mnt/chroot-ubuntu-xenial
sudo debootstrap xenial /mnt/chroot-ubuntu-xenial/ http://archive.ubuntu.com/ubuntu/ # suite target mirror
sudo chroot /mnt/chroot-ubuntu-xenial/ /bin/bash # chroot to guest os
```

#### LXC

Can be used as follows (with `lxc`) to create an unprivileged container:

```sh
cat /etc/subuid # UID map for user
cat /etc/subgid # GID map for user

# Add user to config file to allow for creation of network devices on the host
# that then can be used by the containers
sudo bash -c 'echo user veth lxcbr0 10 >> /etc/lxc/lxc-usernet'

# Create lxc config from template (set permissions to 664)
cp /etc/lxc/default.conf ~/.config/lxc/default.conf

# Add user uid and gid to the template (change out IDs as necessary)
echo lxc.idmap = u 0 231072 65536 >> ~/.config/lxc/default.conf
echo lxc.idmap = g 0 231072 65536 >> ~/.config/lxc/default.conf
```

Access control management (`acl` package) must be used to correctly set permissions for LXC:

```sh
setfacl -R -m u:231072:x ~/.local
```

After that the (unprivileged) container can be created and started and attached to:

````sh
lxc-create \
  --template download \
  --name unpriv-cont-user \
  -- \
  --dist ubuntu \
  --release xenial \
  --arch amd64

lxc-start -n unpriv-cont-user -d

# List containers
lxc-ls -f

# Print container info
lxc-info -n unpriv-cont-user

# Attach to the container
lxc-attach -n unpriv-cont-user

# Stop the container
lxc-stop -n unpriv-cont-user

# Destroy the container
lxc-destroy -n unpriv-cont-user
``

Privileged containers can be created with `sudo lxc-create ...`

#### systemd-nspawn Containers

These can be managed with `systemd-container`:

```sh
sudo debootstrap --arch=amd64 stable ~/DebianContainer # prepares the container
sudo systemd-nspawn -D ~/DebianContainer # creates the container

sudo machinectl list # lists running containers
sudo machinectl terminate DebianContainer # stops the container
````

## Container Standards and Runtimes

Often compared to VMs, but on an OS-level and being bound to the host OS. Can run simple scripts but also full-sized web servers. Containers can run anywhere where a container runtime is implemented (Bare-Metal, Virtual Maching, Cloud, ...).

The Open Container Initiative (OCI) container standard is the most prevalent nowadays, `appc` was used, but is slowly merged into OCI.

Container Runtimes implement these standards to allow for host OSs to run containers. When a runtime runs a container they interact with the kernel, build isolation and more.

### OCI Standard

Specifies the configuration, execution env and lifecycle of containers. It e.g. defines:

- Standard operations (start, stop, create, copy, ...)
- Idempotency
- Infrastructure agnostic (every OCI runtime can every OCI container)
- Container properties (state includes version, id, status, pid, bundle, ...)
- lifecycle events
- And more like error, warning handling, hooks, ...

It also defines the image formate specification including a manifest, index, (filesystem) layout, filesystem layer, image configuration, conversion so that a container can be created from a OCI-compatible image.

And the latest standard the `Distribution Specification` defines an API protocol standardizing how OCI images and other content is distributed (~ artifact distribution).

### `runc` Runtime

- uses `libcontainer` providing a low level runtime focused on container execution
- implements OCI and can create and run OCI containers
- very simple, does not expose an API and no container image management capabilities
- it can build images but does not provide integrity checks and downloading for images
- can be used through systemd

### `containerd` Runtime

- simple, but adds storage and transfer of images and attachment to storage and network as features
- should act as underlying embedded daemon for other systems like Google Cloud, Cloud Foundry, ...
- supports OCI image spec and runtime by using `runc` under the hood
- Can be managed with `crictl`

### Docker Runtime

- Feature rich and complex management platform
- supports container image management to container lifecycle and runtime management
- implements OCI and provides tools to bundle applications into signed OCI container images
- also allows for interaction with image registries and manages container throughout the full lifecycle
- Complex architecture with Docker Engine (a client-server application) running on some Linux distros, MacOS and Windows
- Docker Engine is compose of the Docker host (running the Docker daemon `dockerd`), a REST API and a Docker client (`docker` CLI)
- client can run on the same machine as the docker daemon but does not have to
- the daemon builds, run and distributes docker containers by listening to docker API requests and acting on them
- the daemon manages images, ontainers, networks and volumes and can work in a distributed way with other daemons
- docker registries (like Docker Hub) store images, but custom registries can be added

To manage docker as a non root user, we must do the following:

```sh
sudo groupadd docker
sudo usermod -aG docker $USER
newgrp docker
```

### Podman Runtime

- Allows developing, managing and running OCI containers in privileged and unprivileged mode
- Podman is a daemonless engine providing a REST API service allowing containers to be launched on-demand by remote applications
- uses `libpod` and `conmon` (monitoring, runs in parallel for each container) under the hood
- mimicks the Docker CLI and can manage OCI images and run containers

### CRI-O Runtime

- Minimal implementation of the Container Runtime Interface (CRI) (<- Kubernetes interface for OCI runtimes)
- Lightweight alternative to docker for Kubernetes
- CRI-O is optimized for Kubernetes (implements CNI)
- Packed with libraries that pull container images from registries and create container filesystems
- uses `runc` to run containers (handled by `conmon`)
- Supports additional container security features and can be managed with `crictl`

## Image Operations

- An image is a template for running a container
- It is a tarball with configuration files including container configuration options and runtime settings
- Container runtimes load images to run them as containers, multiple containers can be started from one image
- OCI images can be built by e.g.: `docker build`, `podman build`, `buildah build`
- Container images can be created from scratch, from another image or from a running container
- Image consumes storage as it is a bundle of files but can also consume network bandwidth when downloading/uploading from/to registries
- Container confiugration (base image) is mounted read-only into a container
- Each added feature to the image is added as an additional layer on the top

### Registries and Repositories

- An image registry is an efficient and secure storage option for images that could be setup in the cloud or on-prem
- Public registries are also available
- Examples: Docker Hub, Quay, Amazon ECS, ...
- Image management tools and container runtimes may provide image caching features to save bandwidth

### Container Image Operations

- Typical image operations would allow users to manage the container image lifecycle from creation to storing it in a registry, to preparing it to be run as a container
- Only some runtimes support all image operations, most only a subset

#### runc

- Runs containers bundled in the OCI format based on a JSON config file
- Config file includes host specific settings and might be modified on a different host

#### Docker

Uses the `Dockerfile` as a base, which includes operations that are executed by `dockerd` when building the image. `docker container commit` can be used to create an image from a running conainer. The execution steps in a container build can be shown with `docker image history`.

You can set up a private registry by using a docker image:

```sh
docker run -d -p 5000:5000 --restart=always --name registry registry:2

# Pull from public repository
docker image pull alpine:3.14

# Tag the image
docker image tag alpine:3.14 localhost:5000/myalps

# Push to private registry
docker image push localhost:5000/myalps
```

#### Buildah/Podman

Can create images from scratch or with a Dockerfile. They can be OCI images or docker images in a `Containerfile` which exposes the same operations as Docker.

`podman search` and `podman image $ACTION` can be used to handle the images (CLI very similar to Docker)

#### crictl

Does not support image build operations by itself. It allows limited image management (list, inspect, pull, remove).

## Container Operations

A running container is a process on a host running a container runtime.

### runc

`runc run` to run a container. `runc list` to show running containers. The actual lifecycle (restart, ...) could be managed by systemd.

### Docker

`docker create/start/stop/kill/rm` for basic operations. `docker attach` to access standard I/O or `docker exec` to run commands. `pause`/`unpause` are also available. `docker top` to list processes running in a container.

### Podman

Very similar to docker.

### CRI-O

Can be managed with `crictl` which exposes `create/start/run/stop/rm/logs/stats/ps/attach/exec/inspect`.

## Building Container Images

- All components of a container have to be specified manually or through scripts
- Some of the build tools include steps that need to be performed by the user
- Some configuration can be applied at runtime
- A lot of different tools available (runc, Docker, Buildah, Podman, ...)
- Often a script file that automates part of the build, with the most popular being the Dockerfile
- Docker Engine runs every instruction in a Dockerfile to build an image
- Context is required (path to a set of files, localhost, or URL) and is the basis of the FS of the container
- Typically context only includes files needed for the build plus the Dockerfile itself
- Another example is Podman's Containerfile
- Another option to create the image is to export a running container which tarballs the containers root filesystem

### Instructions

There are build time instructions:

- FROM: Defines the base image
- ARG: Can be used in build command (`docker build --build-arg ARG1=1`), default can be set in the Dockerfile
- RUN: Run command inside the intermediate container that builds the final image
- LABEL: Adds metadata to the resulting image in key value format
- EXPOSE: Exposes network ports to open from the container
- COPY: To copy content from the build context to the intermediate container that gets committed to the resulting image
- ADD: Like copy, but can be used with URLs and automatically extracts tarfiles
- WORKDIR: Sets base directory for other commands like `RUN`, `CMD`, ...
- ENV: Sets env variables inside the container
- VOLUME: Create a mount point and mount external storage to it
- USER: To set the user name or UID of the resulting image for some commands
- ONBUILD: Commands to be executed at a later time (when the image becomes the base image of other images)
- STOPSIGNAL: Set up a system call signal (e.g. for custom exit signals)
- HEALTHCHECK: A command can be defined to check if the application is still healthy, if not the container can be stopped
- SHELL: Defines the default shell for instructions (otherwise on Linux defaults to `bin/sh -c`)

There are also runtime instructions that are executed when the container is started:

- CMD: Only one can be present in a Dockerfile (preferred form: `CMD ["nginx", "-g", "daemon off"]`)
- ENTRYPOINT: A `CMD` that cannot be overriden at runtime, only its arguments

## Networking

- A typical (physical or virtual) machine is connected to at least one network to form a larger computing env
- Networking is governed by networking standards, principles, topologies, protocols, policies and rules
- Containers encapsulate running applications, so these have to be (generally) attached to the networks
- The two important container networking standards are: Container Network Model (CNM) and Container Network Interface (CNI)

### Container Network Model

- Introduced by Docker and implemented by `libnetwork`
- CNM specifies a set of objects for a container to join a network
- It defines a network sandbox for each container (= isolated networking stack inside the container)
- Containers are reachable via endpoints (= an interface paired with an interface on a network)
- Multiple endpoints per sandbox are supported so that containers can access multiple networks simultaneously
- `libnetwork` introduces a consistent programming interface to abstract applications network connectivity
- Different network drivers are supported by `libnetwork`:
  - `bridge` (= Linux bridge)
  - `overlay` (= a network spanning multiple hosts with VXLAN)
  - `null` (= no networking)
  - `remote` (= custom network plugins)

### Container Network Interface

- Simpler specification with accompanying libraries, focusing only on the networking connectivity of a container
- Used in Kubernetes
- There are tree different plugin groups:
  - Main: Includes plugins to create `bridge`, `ipvlan`, `loopback`, `ptp` (for `veth`), `macvlan`, `host-device` and more to create new interfaces
  - IPAM: `dhcp`, `host-local` (DB of IP addresses), `static` (static IPv4/6) plugins for IP address allocation
  - Meta: e.g. `flannel`, `bandwidth`, `firewall`, `sbr` (source-based routing), `portmap`, systctl `tuning`

### Container Networking in different technologies

- `runc`: Does not provide management capabilities but allows for setting up a network stack

#### Docker

anages traffic with idtables and server routing rules and additional features (packet encapsulation, encryption, routing mesh, session affinity)

Docker Networking Drivers are:

- `Bridge`:
  - Default network type
  - Isolates container network from host and acts as DHCP server assigning unique IP addresses to containers
  - Useful if all containers should talk to each other
  - Can be user-defined or default
  - User-defined allows for traffic isolation, automatic DNS resolution, on-the-fly container (dis)connection and sharing env variables between containers
- `Host`:
  - No network isolation between the container and the host, container can directly access the host network
  - Eliminates the need for NAT or proxying
- `Overlay`:
  - Spans multiple hosts (typically part of a cluster)
  - Allows container traffic to be routed between hosts in e.g. a larger cluster
  - Used in container orchestration platforms for cluster-wide communication
  - Allows for traffic isolation and mangement through rules and policies
- `Macvlan`:
  - Changes appearance of the container on a physical network
  - Container appears as a physical device with its own MAC address
  - Container is directly connected to a physical network
- `None`:
  - Disables networking of a container altogether
  - The container can still use third-party network drivers with this option enabled
- `Network Plugins`:
  - Third-party network drivers to integrate Docker with specialized network stacks

There are also Docker netowrk modes to manage network capabilitie in multi-host clusters (e.g. Docker Swarm):

- `Internal Mode`:
  - Isolates an overlay network if we want to restrict container access to the outside world
- `Ingress Mode`:
  - Networking of swarms by implementing a routing mesh across hosts of a swarm cluster
  - Only one ingress can be defined

Docker Network can be managed with `docker network`. `docker network ls` lists all networks available. `docker network inspect NAME` shows information about the network (e.g. driver, subnet, gateway, ...).

Listing all containers attached to a network can be done with `docker network inspect <network-name>|grep Name|tail -n
+2|cut -d':' -f2|tr -d ' ,"'`.

New networks can be created wit `docker network create NAME`. A docker container can be started without a network with e.g. `docker container run -it --network=none alpine sh`. Disconnect a container from a network with: `docker network disconnect`.

#### Podman

Podman supports CNI and provides two networking drivers:

- `Bridge`:
  - Default networking type, where the container network is isolated from the host network
  - The user can define a network subnet, IP range and gateway
- `Macvlan`:
  - To directly connect a container to a physical network
  - In rootless mode a separate network namespace has to be created
  - `dhcp` has to be enabled (or a DHCP client has to be included) for the container to interact with the host network

A custom network with a specific subnet and gateway can be created with: `podman network create --subnet 10.99.3.0/24 --gateway 10.99.3.3 mynet`. Running a container with access to this network can be achieved with: `podman container run -d --name mywebmynet --net mynet nginx`.

### CRI-O

- Networking with Kubernetes pods in mind, based on CNI
- Allows for containers to connect to a network when created and removed when the pods are destroyed
- When starting with `crio` there are options to specify host netwrok IP and the network namespace

## Container Storage

### Docker

- Very complex methods of handling storage (different concepts, access modes, storage drivers and volume types)
- UnionFS is used to overlay a base image with storage layers (ephemeral, custom, config layers):
- Ephemeral Storage: Reserved for I/O, not recommeded for persistent storage
- Due to the Copy-On-Write strategy read-only base container image files can be modified, these are copied to the ephemeral storage layer as well, the user from then on interacts with the copy of the file
- Storage drivers need to be used to write to the containers writable layer (`overlay`, `overlay2`, `aufs`, supporting `xfs` and `ext4`; `devicemapper` with `direct-lvm` and `vfs` for any filesystem)
- Ephemeral storage gets deleted with the container
- For persistent storage a volume should be mounted on the container
- Persistent volumes are not managed by UnionFS and do not get deleted together with a container
- Docker uses volume plugins to store Docker volumes on remote hosts, for encryption and other functionality

Run a container with a mounted volume: `docker container run -ti --mount target=/data --name cvol alpine sh`. List all volumes with `docker volume ls`. Create a named volume with: `docker volume create --name myvol`

Mount a host directory inside a container: `docker container run -ti --mount type=bind,source=/mnt/shared,target=/data{,readonly} --name csharedvol alpine sh` (omit readonly for writable mount).

Display disk usage with: `docker system df` and remove unused volumes with `docker volume prune`.

### Podman

- Uses `containers/storage` library to manage image, container storage and storage layers
- A layer is a CoW filesystem defined by a set of changes to its parent
- Any layer can have only one parent, but one layer can be a parent to multiple layers
- Podman also uses drivers to implement storage (`overlayfs`, `AUFS`, `btrfs`, `zfs`)
- The default driver is a local driver, other storage plugins have to be defined in a special configuration file
- Local storage volumes allows for mounting a directory on the host with its options resembling Linux `mount`

Podman commands are very similar to the Docker commands.

### CRI-O

- Also uses `containers/storage` with CoW and largely the same storage drivers as Podman
- `overlayfs` as the default storage driver

## Runtime and Container Security

- Environment: Surrounding hardware, OS, hypervisors, network fabric, storage services (each with their own considerations)
- Container Runtime:
  - Container processes inherit permissions from the user running the engine
  - Best option is to not run the runtime with root privileges (Podman/Docker rootless)
- Client access:
  - Client should access the runtime in a secure way
  - Either through SSH or TLS over HTTPS
  - Calls to the daemon can be secured by default to ensure that every call is verified
  - Other client authentication mechanisms are possible
- Container Images:
  - Use only trusted images and image sources
  - Verify with content trust signature verification (cross-runtime verification might not work everywhere)
- Secure Containers:
  - Run in rootless mode if at all possible
  - Otherwise `capabilities` can be used to give root access but limit operations at a more granular level
  - Ensure that only strictly necessary permissions are given
- Security Tools:
  - Scanning software can be used for images to ensure that they are safe
  - AppArmor can be used to to secure container processes
  - Seccomp (linux kernel feature to limit process access to host resources) to allow/block system calls, kernel memory, kernel I/O, namespaces, ...
