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
- Most installations use `systemd` to run the kubelet process on each node
- Compiling K8s from source is also possible to modify aspects of components

### Installation Process

1. Update system packages
2. Install necessary packages (Ubuntu: `apt-transport-https`, `tree`, `software-properties-common`, `ca-certificates`, `socat`)
3. Disable swap
4. Enable kernel modules: `overlay`, `br_netfilter`
5. Update kernel networking to allow for traffic and apply changes with `sysctl --system`
6. Install necessary key for software
7. Install containerd with default config (`containerd config default | tee /etc/containerd/config.toml`) and SystemDCgroup `sed -e 's/SystemdCgroup = false/SystemdCgroup = true/g' -i /etc/containerd/config.toml`
8. Download public signing key for Kubernetes and add the repository
9. Install Kubernetes (`apt install -y kubeadm=1.33.1-1.1 kubelet=1.33.1-1.1 kubectl=1.33.1-1.1`)
10. Pin the packages: `apt-mark hold kubelet kubeadm kubectl`
11. Persist the hostname for later use: `hostname -i` and `ip addr show`
12. Add local DNS aliases for the control plane with the IP in step 11 (`$IP cp` as new line in /etc/hosts)
13. Create a configuration file to initialize the cluster (different for `containerd`, `Docker`, `cri-o`)
14. Initialize the control plane (`kubeadm init --config=kubeadm-config.yaml --upload-certs --node-name=cp | tee kubeadm-init.out`)
15. Allow non-root admin access to the cluster (`sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config` <- Set permissions for this to the current user)
16. Decide on the networking plugin (Cilium in this case for Network Policies, make sure to init with CNI)

Command for step 5:

```sh
cat << EOF | tee /etc/sysctl.d/kubernetes.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF
```

Command for step 6:

```sh
mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
| sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

Example file for step 13:

```yaml
apiVersion: kubeadm.k8s.io/v1beta4
kind: ClusterConfiguration
kubernetesVersion: 1.33.1 #<-- Use the word stable for newest version
controlPlaneEndpoint: "k8scp:6443" #<-- Use the alias we put in /etc/hosts not the IP
networking:
  podSubnet: 192.168.0.0/16
```

### Growing a Cluster / Adding Worker Nodes

1. Install necessary packages, turn swap off, enable kernel modules, install containerd and K8s (see previous chapter)
2. On the control plane node, generate up a join command: `sudo kubeadm token create --print-join-command`
3. Copy kubernetes config to worker node
4. Execute join command on the worker node with `--node-name=worker` appended

### Finish Cluster Setup

1. Show nodes `kubectl get node` and `kubectl describe node cp`
2. Check if networking pods and others are running `get pods --all-namespaces`
3. Check if network interfaces were created `ip a` should show e.g. `cilium` interfaces

### Deploy a simple application

1. Create a deployment: `kubectl create deployment nginx --image=nginx`
2. Store a command into a file: `kubectl get deployment nginx --dry-run=client -oyaml > first.yaml`
3. Set `containerPort` and `protocol` in `containers.ports`
4. Replace (or create) the existing deployment: `kubectl replace -f first.yaml --force`
5. Create a service by exposing: `kubectl expose deployment/nginx`
6. Verify the service: `kubectl get svc nginx` and the endpoint: `kubectl get ep`
7. Optional: Scale up the service: `kubectl scale deployment nginx --replicas=3`

### Access from outside the cluster

1. Check existing Kubernetes config inside pod: `kubetcl exec POD_NAME -- printenv |grep KUBERNETES`
2. Expose with load balancer: `kubectl expose deployment nginx --type=LoadBalancer`
3. If not needed delete BOTH deployment and service

## Kubernetes Architecture

### Basic Node Maintenance

#### Backing Up etcd with etcdctl

1. Find the data directory for etcd: `sudo grep data-dir /etc/kubernetes/manifests/etcd.yaml`
2. Enter the `etcd` container: `kubectl -n kube-system exec -it etcd-<TAB> -- sh`
3. Get the necessary file paths for TLS from `/etc/kubernetes/pki/etcd`
4. Check `etcd` health by providing the paths from step 3
5. Check the number of databases in the cluster
6. Save by using built-in snapshot utility
7. Verify the existence of the backup: `sudo ls -l /var/lib/etcd/`
8. Store the backup together with other relevant information (kubeadm-config and pki/etcd)

Command for step 4:

```sh
kubectl -n kube-system exec -it etcd-cp -- sh \
-c "ETCDCTL_API=3 \
ETCDCTL_CACERT=/etc/kubernetes/pki/etcd/ca.crt \
ETCDCTL_CERT=/etc/kubernetes/pki/etcd/server.crt \
ETCDCTL_KEY=/etc/kubernetes/pki/etcd/server.key \
etcdctl endpoint health"
```

Command for step 5:

```sh
kubectl -n kube-system exec -it etcd-cp -- sh \
-c "ETCDCTL_API=3 \
ETCDCTL_CACERT=/etc/kubernetes/pki/etcd/ca.crt \
ETCDCTL_CERT=/etc/kubernetes/pki/etcd/server.crt \
ETCDCTL_KEY=/etc/kubernetes/pki/etcd/server.key \
etcdctl --endpoints=https://127.0.0.1:2379 member list -w table"
```

Command for step 6:

```sh
kubectl -n kube-system exec -it etcd-cp -- sh \
-c "ETCDCTL_API=3 \
ETCDCTL_CACERT=/etc/kubernetes/pki/etcd/ca.crt \
ETCDCTL_CERT=/etc/kubernetes/pki/etcd/server.crt \
ETCDCTL_KEY=/etc/kubernetes/pki/etcd/server.key \
etcdctl --endpoints=https://127.0.0.1:2379 snapshot save /var/lib/etcd/snapshot.db "
```

Script for step 8:

```sh
mkdir $HOME/backup
sudo cp /var/lib/etcd/snapshot.db $HOME/backup/snapshot.db-$(date +%m-%d-%y)
sudo cp /root/kubeadm-config.yaml $HOME/backup/
sudo cp -r /etc/kubernetes/pki/etcd $HOME/backup/
```

#### Upgrade the cluster

Start with the control plane node(s)

1. Remove the hold on `kubeadm`: `sudo apt-mark unhold kubeadm`
2. Install new version: `sudo apt install -y kubeadm=1.34.1-1.1`
3. Reinstate the hold: `sudo apt-mark hold kubeadm`
4. Verify the version: `sudo kubeadm version`
5. Evict as many pods as possible (everything but DaemonSets): `kubectl drain cp --ignore-daemonsets`
6. Plan/prepare the `kubeadm` upgrade command: `sudo kubeadm upgrade plan`
7. Actually upgrade: `sudo kubeadm upgrade apply v1.34.1`
8. Check the node status: `kubectl get node`
9. Release the hold on `kubelet` and `kubectl`: `sudo apt-mark unhold kubelet kubectl`
10. Upgrade `kubelet` and `kubectl` to the same version as `kubeadm`: `sudo apt install -y kubelet=1.34.1-1.1 kubectl=1.34.1-1.1`
11. Restart the daemons: `sudo systemctl daemon-reload && sudo systemctl restart kubelet`
12. Verify the cp node update: `kubectl get node`
13. Make the cp node available for scheduling again: `kubectl uncordon cp`
14. Verify the Ready status of the node `kubectl get node`

Then move on to the worker nodes:

1. Remove the hold: `sudo apt-mark unhold kubeadm`
2. Check available versions: `sudo apt-cache madison kubeadm`
3. Install the new version: `sudo apt update && sudo apt install -y kubeadm=1.34.2-1.1`
4. Add the hold: `sudo apt-mark hold kubeadm`
5. Drain the worker node FROM THE CONTROL PLANE NODE: `kubectl drain worker --ignore-daemonsets`
6. BACK TO WORKER NODE, start upgrade: `sudo kubeadm upgrade node`
7. Remove hold on kubelet and kubectl, install updates and add hold again
8. Restart daemons (see step 11 of cp update)
9. CONTROL PLANE NODE, verify status of worker: `kubectl get node`
10. Allow pods to be deployed to the node again: `kubectl uncordon worker`
11. Verify the status of the nodes: `kubectl get nodes` (both `Ready`, no `SchedulingDisabled`)

### Working with CPU and memory constraints

Use `kubectl logs POD` and `top` to monitor nodes and pods. Use `kubectl get events` to see if pods get evicted.

1. Get information on resource config for container: `kubectl get deployment hog -oyaml` (`spec.containers.resources`)
2. Adapt yaml (Create if not present: `kubectl get deployment hog -o yaml > hog.yaml`)
3. Limit by applying resource limits
4. Replace the deployment: `kubectl replace -f hog.yaml`
5. Delete deployment if problems arise: `kubectl delete deployment hog`
6. Change parameters and restart

File contents for step 3:

```yaml
imagePullPolicy: Always
name: hog
resources: # Edit to remove {}
  limits: # Add these 4 lines
    memory: "4Gi"
  requests:
    memory: "2500Mi"
terminationMessagePath: /dev/termination-log
terminationMessagePolicy: File
```

### Resource Limits in Namespaces

You can set resource limits with the `LimitRange` object:

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: low-resource-range
spec:
  limits:
    - default:
        cpu: 1
        memory: 500Mi
      defaultRequest:
        cpu: 0.5
        memory: 100Mi
      type: Container
```

You can use this object in a namespace with: `kubectl create -f low-resource-range.yaml -n low-usage-limit` and show it with: `kubectl get limitrange`. For pods in that namespace use `kubectl get pod POD -oyaml` to see that they inherit the options.

## APIs and Access

You can impersonate other users by using `--as` with your CLI commands.

### Configuring TLS

Prerequisites: Certificate data in `$HOME/.kube/config`. Store in environment variables with: `export client=$(yq .users.0.user.client-certificate-data $HOME/.kube/config)`, `export key=$(yq .users.0.user.client-key-data $HOME/.kube/config)` and `export auth=$(yq .clusters.0.cluster.certificate-authority-data $HOME/.kube/config)`.

1. Encode the keys: `echo $client | base64 -d - > ./client.pem`, `echo $key | base64 -d - > ./client.pem`, `echo $auth | base64 -d - > ./client.pem`
2. Get the server URL: `kubectl config view | grep server`
3. Make requests with the values: `curl --cert ./client.pem --key ./client-key.pem --cacert ./ca.pem https://k8scp:6443/api/v1/pods`

### Exploring API Calls

We can use `strace` to view what a command does, e.g. `strace kubectl get endpoints`. There should be `openat` calls using something like `.kube/cache/discovery/...`. This directory contains a lot of configuration information for Kubernetes.

Inside the directory, you can list all objects in a specific file: `python3 -m json.tool v1/serverresources.json | grep kind` (there might be more in other files)

## Working with API Objects

### REST API Access

1. Store the token for accessing the API in an env variable: `export token=$(kubectl create token default)`
2. Call the API with e.g.: `curl https://k8scp:6443/apis --header "Authorization: Bearer $token" -k`
3. Try different endpoints: `curl https://k8scp:6443/api/v1/namespaces --header "Authorization: Bearer $token" -k`

### Using the proxy
