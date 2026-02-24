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
