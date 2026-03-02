# Kubernetes Fundamentals

## Installing and Configuring

### Kubeadm

1. Run `kubeadm` on the control plane node
2. Install a networking plugin that satisfies IP-per-Pod (Calico, Flannel, Cilium, ...)
3. Run `kubeadm join` on each worker node using token+hash from step 1

#### Upgrade

`kubeadm upgrade` can be used. Most users stay only on major releases. Subcommands are:

- `plan` to check current version against installed version
- `apply` to apply an upgrade to the first control plane node
- `diff` to preview changes (`apply --dry-run`)
- `node` to update local kubelet configuration on worker nodes or seconday control plane nodes

Upgrade workflow is as follows (Make sure to upgrade one control plane node, confirm and then move to worker nodes):

1. Update system software and K8s packages
2. Check against running K8s version
3. Drain the control plane node
4. Review the planned upgrade
5. Apply the upgrade to the control plane node
6. Uncordon the control plane node
7. Repeat node-specific upgrades for control planes and worker nodes and restart kubelets

### Installing a Pod Network

Make sure to do the following before initializing a cluster:

1. Plan network configuration to avoid IP address conflicts
2. Choose a Pod networking solution that aligns with deployment needs

#### Calico

- Flat layer 3 networking model without IP encapsulation (single IPs, no overhead by wrapping packets)
- Underlying network must support routing pod IPs to the hosts (Calico can be configured to encapsulate though)

#### Flannel

- Layer-3 IPv4 overlay network for inter-node communication
- Simple option suited for traffic between nodes rather than intra-node or container-level networking
- Supports multiple backend mechanisms (e.g. VXLAN)
- Each node runs a daemon `flanneld` that assigns and manages subnet leases
- Recommended to configure it before deploying any pods

#### Kube-Router

- Combines networking, routing, firewalling, load-balancing
- Aims to provide operational simplicity by not requiring multiple tools for the features above
- Uses IPVS/LVS as a service proxy, BGP (GoBGP library) for Pod networking
- Supports Network Policy semantics
- BGP for inter-node Pod networking and IPVS for load balanced proxy services

#### Cilium

- Leverages eBPF for fine-grained network control and observability
- Can serve as a service mesh
- Features include identity-aware security, load balancing, transparent encryption

### Other Installation Tools

- `Kubespray` Ansible playbook
- `kops` CLI TOOL (primary for AWS)
- `KIND` Kubernetes in Docker to run clusters locally using Docker containers as nodes

### Installation considerations

- Infrastructure Provider: Public, private, on-premise? Physical or virtual nodes?
- OS: Debian, Ubuntu OR container-optimized Fedora CoreOS OR Nix
- Networking: Which plugin? Do we need an overlay for pod-to-pod communication
- etcd: External, colocated or embedded?
- HA: Do we need highly available control plane nodes? How are failover/redundancy strategies implemented?
