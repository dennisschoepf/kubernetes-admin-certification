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
