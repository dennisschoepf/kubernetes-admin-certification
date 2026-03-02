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
