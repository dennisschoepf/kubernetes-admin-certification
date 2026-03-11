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

### Stateful Sets

Object designed for applications requiring stable identities and persistent state (e.g. databases). Each pod in a StatefulSet is assigned an identity that is composed of:

- Stable network identity: DNS name that does not change
- Stable storage: Pods can be associated with persistent volumes that remain consistent across restarts
- Ordinal index: Pods are numbered sequentially and the index is tied to the identity

The identity persists regardless of the node the pod is running on. The deployment order is sequential (`app-0` has to start before `app-1`, ...)

### Autoscaling

The HPA (horizontal pod autoscaler) dynamically adjusts the number of pod replicas in a deployment, replica set or stateful set based on resource utilization or other custom metrics. By default it targets 80% CPU utilization, but you can configure custom metrics via the Kubernetes metrics server.

### RBAC

Securing access to resources in a cluster can be done with the `rbac.authorization.k8s.io/v1` API group API group. It has four stable resources:

- Role (A set of permissions that apply to resources in a namespace)
- ClusterRole (Similar to role, but cluster wide)
- RoleBinding (Grants permissions defined in a Role to a user, group or ServiceAccount)
- ClusterRoleBinding (~ RoleBinding for Cluster)

Example applications:

- A role that allows a developer to only read pods in a certain namespace
- A cluster role permitting an admin to create deployments in any namespace
- A rolebinding to associate these permissions to a specific user or service account

### REST API Access

1. Store the token for accessing the API in an env variable: `export token=$(kubectl create token default)`
2. Call the API with e.g.: `curl https://k8scp:6443/apis --header "Authorization: Bearer $token" -k`
3. Try different endpoints: `curl https://k8scp:6443/api/v1/namespaces --header "Authorization: Bearer $token" -k`

### Using the Proxy

Using the Proxy is another way to interact with the API. It can be run from within a Pod or through a sidecar. You can start it with: `kubectl proxy --api-prefix=/ &`. Now you can just `curl http://127.0.0.1:8001/api`.

### Working with Jobs

Create a Job like this with `kubectl create -f job.yaml`:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: sleepy
spec:
  template:
    spec:
      containers:
        - name: resting
          image: busybox
          command: ["/bin/sleep"]
          args: ["3"]
      restartPolicy: Never
```

Show running jobs with: `kubectl get job` and specific info about a job with `kubectl describe jobs.batch NAME`. You can delete a job with: `kubectl delete job.batch NAME`. To run a job multiple times use `template.spec.completions = NUMBER`. Show completions status with: `kubectl get jobs.batch`. To run multiple jobs in parallel use `template.spec.parallelism = NUMBER`.

`activeDeadlineSeconds` can be used in the YAML to set a maximum runtime of the job. If a job exceeds it `kubectl get job sleepy -o yaml` should show something like this:

```yaml
status:
  conditions:
    - lastProbeTime: 2024-08-23T10:48:002
      lastTransitionTime: 2024-08-23T10:48:002
      message: Job was active longer than specified deadline
      reason: DeadlineExceeded
      status: "True"
      type: Failed
  failed: 2
  startTime: 2024-08-23T10:48:002
  succeeded: 3
```

### CronJobs

CronJobs can be configured like this:

```yaml
apiVersion: batch/v1
kind: CronJob #<-- Update this line to CronJob
metadata:
  name: sleepy
spec:
  schedule: "*/2 * * * *" #<-- All Linux style cronjob syntax
  jobTemplate: #<-- Use jobTemplate and spec: core
    spec:
      template: #<-- This and following lines move
        spec: #<-- four spaces to the right
          containers:
            - name: resting
              image: busybox
              command: ["/bin/sleep"]
              args: ["3"]
          restartPolicy: Never
```

List cronjobs with `kubectl get cronjobs.batch`. Similar to a Job you can use `activeDeadlineSeconds` to limit its maximum runtime.

## Helm and Kustomize

Real-world applications are not as straight-forward as we experienced in the lab exercices. Helm addresses some of these shortcomings by:

- Providing a way to manage Kubernetes packages
- And to manage multiple interdependent components

A Helm chart is a package that includes all manifests, templates and scripts for installation, configuration and removal of a package. The package (a compressed tarball) can consist of Secrets, Ingress, Deployments, ... so that they do not have to be handled individually. This allows for deploying a full application quickly.

Charts can be shared publicly and privately, are version controlled, can be customized (values.yaml e.g. for different environments) and integrated into pipelines. You can also revert an application to a known good state without restoring multiple YAML files and there is a large pre-existing body of charts available.

### Chart Structure

A chart typically looks like this (for templating - Go templating is used):

```
├── Chart.yaml
├── README.md
├── templates
│   ├── NOTES.txt
│   ├── _helpers.tpl
│   ├── configmap.yaml
│   ├── deployment.yaml
│   ├── pvc.yaml
│   ├── secrets.yaml
│   └── svc.yaml
└── values.yaml
```

This is how a Helm template for a Kubernetes secret could look like:

```go
apiVersion: v1
kind: Secret
metadata:
  name: { { template "fullname" . } }
  labels:
    app: { { template "fullname" . } }
    chart: "{{ .Chart.Name }}-{{ .Chart.Version }}"
    release: "{{ .Release.Name }}"
    heritage: "{{ .Release.Service }}"
type: Opaque
data:
  mariadb-root-password:
    { { default "" .Values.mariadbRootPassword | b64enc | quote } }
  mariadb-password: { { default "" .Values.mariadbPassword | b64enc | quote } }
```

Helm Chart can be distributed through repositories that store two things: `index.yaml` with all available charts and metadata and the actual chart packages stored as compressed tarballs.

`Artifact Hub` is a public catalog that aggregates charts from a number of repositories, it can be searched with e.g. `helm search hub redis`.

A repository can be added with: `$ helm repo add bitnami ht‌tps://charts.bitnami.com/bitnami` and a specific repo can be searched with: `$ helm search repo bitnami/redis`.

### Deploying a chart

The `README.md` typically includes documentation on dependencies, options and external resources. The Readme can be accessed by pulling the chart: `helm pull bitnami/apache --untar` and inspecting its contents. If all prerequisites are met, you can install with: `$ helm install anotherweb .`.

The cahrt release can be managed by using commands to list, upgrade, rollback or delete it.

### Kustomize

A K8s config management tool that should simplify the customization of YAML manifests. Instead of relying on text substitution, Kustomize enables you to define a base set of resources and customize them for different environments with `overlays`. Overlays could modify image tags, labels or resource limits.

Kustomize is included in `kubectl` and can be used with `kubectl kustomize`. Changes done by kustomize can be applied with: `$ kubectl apply -k ./dir`. Configuration is done in `kustomization.yaml` files. They define the resources to manage and the modifications that should be applied.

Kustomize splits config into two parts:

- Bases (Directories that contain shared, common config, e.g. a deployment definition for all envs)
- Overlays (Allow for environment-specific changes, e.g. different replica counts, image tags on different envs)

In addition Kustomize supports different mechanisms:

- Patches: Allow fine-grained changes to specific resources (e.g. adding an env variable, modifying a field)
- Transformers: Broad modifications across multiple resources, e.g. injecting a label to every object in a deployment
- Generators: Create resources dynamically (like ConfigMaps and secrets from other files)

### Working with Helm and Charts (Lab)

Install helm by downloading the helm tarball, extract it and move to a location in the path in the path. Try to search the hub afterwards: `helm search hub database`.

You can also add other repos right away, e.g.: `helm repo add ealenn https://ealenn.github.io/charts && helm repo update`. You could then install charts with this command: `helm upgrade -i tester ealenn/echo-server --debug`. A pod should have been started.

The chart history can be accessed with `helm list`. Uninstall a chart with `helm uninstall tester`.

You can inspect the tarball for the chart by finding it first (somewhere in `~/.cache/helm/repository`) and then extracting its contents.

To change the values of a helm chart before deploying, do the following:

1. `helm pull CHART --untar`
2. `cd CHART && vim values.yaml`
3. Adapt some values
4. Install the local chart contents with `helm install CHART . -n kube-system`

### Working with HPAs

A deployment for which we can use a horizontal pod autoscaler could look like this:

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hpa-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: hpa-app
  template:
    metadata:
      labels:
        app: hpa-app
    spec:
      containers:
      - name: web
        image: registry.k8s.io/hpa-example
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 5m
          limits:
            cpu: 10m
---
apiVersion: v1
kind: Service
metadata:
  name: hpa-app
spec:
  type: ClusterIP
  selector:
    app: hpa-app
  ports:
  - port: 80
    targetPort: 80
```

You can create an autoscaler object with:

```sh
kubectl autoscale deployment hpa-app \
  --cpu=50% --min=2 --max=10 \
  --dry-run=client -o yaml > hpa.yaml
```

and apply it with `kubectl apply -f hpa.yaml`. Get live updates of how the replicas behave with: `watch kubectl get hpa` and the number of pods increasing with: `watch kubectl get pods -o wide`.

Caution!:

- The `metrics-server` has to be running for HPA to work
- A `cpu` request has to be defined for the container to compute utilization
- `kubectl top pods` can be used to view live CPU usage
- Scaling behavior can be tuned in HPA v2 using custom metrics and stabilization windows

### Working with Kustomize

Set up base yamls (e.g. for deployments and services) in one directory and overlays for different envs in others (`overlay/dev/deployment-patch.yaml`, ...). Add a kustomization.yaml for the configurable values:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namePrefix: lf-
resources:
  - deployment.yaml
  - service.yaml
labels:
  - includeSelectors: true
pairs:
  company: linux-foundation
```

You can then build the configuration with `kubectl kustomize myapp/base` and `kubectl kustomize myapp/overlays/dev` and so on. After building apply it with: `kubectl apply -k myapp/base/`. You can start resources for the other envs by using `kubectl apply -k myapp/overlays/dev`. After that pods should be running for both the base as well as the specific env.

## Kubernetes Storage

A Pod can declare one ore more volumes and specifies where these volumes are mounted within its containers. A volume requires three attributes:

1. Name
2. Type
3. Mount Point

### Local & Ephemeral Values

- `emptyDir` creates a temporary directory for the lifetime of the pod for e.g. scratch space or caching
- `hostPath` mounts a file/directory/device from the host node into the pod e.g. for logs or config files
  - `DirectoryOrCreate` or `FileOrCreate` allows K8s to create resources on demand
  - Caution! Tightly coupled to the node, not portable across clusters, potential security risks

### Cloud Based Volumes

There are provider specific backends for e.g. Google persistent disks, Amazon EBS, Azure File to mount cloud-based block storage (provided that accounts/permissions are configured).

### Network Based Volumes

If multiple pods or nodes need to access the same data, `nfs` and `iscsi` (network attached block storage) can be used.

### Other Volume Types

- `downwardApi` to expose pod metadata as files
- `secret` for Kubernetes secrets
- `configMap` to mount configuration data from e.g. settings files
- `projected` to combine multiple sources into a single volume
- `local` to mount a local disk or partition (e.g. to mount a specific high-performance SSD)
- `fc` for different providers
- `persistentVolumeClaim` to access PersistentVolume
- `csi` to access everything that supports the container storage interface driver

### Creating a ConfigMap

```sh
kubectl create configmap colors \
    --from-literal=text=black \ # Pass k/v pairs right within the command
    --from-file=./favorite \ # Create from file contents
    --from-file=./primary/ # Or from a directory
```

The data is then represented in the configmap `kubectl get configmap colors -oyaml` in the `data` property. A pod can use the configmap by specifying: `spec.containers.env.valueFrom.configMapKeyRef` (specify name and key for the config map).

The values from the file can be included as environment variables with: `spec.containers.envFrom.configMapRef`.

You could also create a YAMl file that describes the key/values in the file itself:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fast-car
  namespace: default
data:
  car.make: Ford
  car.model: Mustang
  car.trim: Shelby
```

This can be included in a pod with: `volumes.configMap.name`.

### Creating a NFS volume

Prerequisites: A NFS somewhere has to exist already.

This file shows how to mount a NFS into a pod:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pvvol-1
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  nfs:
    path: /opt/sfw
    server: cp #<-- Hostname of cp node
    readOnly: false
```

### Creating a PersistentVolumeClaim

Create a pvc with something like this:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-one
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 200Mi
```

In a pod you can mount the claim by specifying it in a yaml file where the claim name matches the name of the PersistentVolumeClaim object:

```yaml
#...
volumes:
  - name: nfs-vol
    persistentVolumeClaim:
      claimName: pvc-one
#...
```

#### Using ResourceQuota to limit PVC usage

A resource quota can be set up like this:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: storagequota
spec:
  hard:
    persistentvolumeclaims: "10"
    requests.storage: "500Mi"
```

When creating this inside a namespace (e.g. `kubectl -n small create -f storage-quota.yaml`) it is applied to the namespace (check with `kubectl describe ns small`). Every claim by a pod (or something else) will increment the claim counter and used storage in `describe ns`. The size of e.g. a NFS share is not counted against the resource quota inside a namespace.

When deleting a persistentVolumeClaim `ReclaimPolicy` is used to act on the underlying storage. It can be one of `Delete`, `Retain`. Manually created PVCs by default are `Retain`, meaning the files still exist after deleting the PVC. `Delete` cant always be used. The volume type does have to support the deleter volume plugin (`NFS` does not for example).

If trying to create a claim that would exceed the resource quota you will get an error `exceeded quota`.

### Creating StorageClasses to dynamically provision volumes

StorageClasses automate the process of manually creating PVs and PVCs. When creating a PVC and specifiying a StorageClass the system automatically creates a PV that meets the requirements.

StorageClasses require a provisioner. For NFS e.g. this: `helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/`. Installing this would also create a storage class for us `kubectl get sc`.

The storage class can then be used in a PVC by specifiying `spec.storageClassName`:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-two
spec:
  storageClassName: nfs-client
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 200Mi
```

This would create both a PV and PVC `kubectl get pv,pvc`. Inside a pod the the persistentVolumeClaim can be used similarly to the previous steps by specifiying `columes.persistentVolumeClaim.name`.

## Services

Services are API resources that define a logical set of pods and a policy for accessing them (achieved through labels and selectors). Services map to Endpoints (~EndpointSlices) to manage traffic to pods.

Rules defined by Services are acted upon by `kube-proxy`, which monitors the API for service and endpoint changes and configures networking rules (with `iptables` or `IPVS` in modern clusters) accordingly, by:

1. Assigning a random IP for a service
2. Listen to traffic to the Services `ClusterIP:Port`
3. Redirecting traffic to the Pods defined in the Service Selectors

Services provide automatic stateless load balancing for matched pods, distributed evenly. Session Affinity can be enabled with the `sessionAffinity` field to ensure that requests from the same client IP are routed to the same Pod (for stateful applications). Headless services are also possible, allowing direct access to individual pod IPs (e.g. for databases).

Version-specific labels can be used to route traffic precisely on breaking changes for updates.

The most common Service Types are:

1. ClusterIP (default) which assigns a stable internal IP to a service, external access possible by combining it with other mechanisms
2. NodePort exposes an application on a specific port of **every** node in the cluster, accessible via `NodeIp:NodePort` (builds upon ClusterIp, by redirecting node traffic on a port to a specific pod ClusterIP)
3. LoadBalancer can utilize an external load balancer to distribute traffic, if no load balancer is available in the environment this falls back to NodePort
4. ExternalName creates a DNS alias (CNAME) that points to an external service outside of the proxy at the DNS level, useful if services live outside of the cluster (e.g. legacy database)

Services are coordinated with several components:

- `kube-controller-manager` creates and manages Service objects
- `kube-apiserver` communicates changes to the networking plugin and to the `kube-proxy` agents on the nodes
- `EndpointSlice`s are used together with networking rules to ensure traffic is routed to the correct pod. These slices track the ephemeral IP addresses of Pods matching the Service selector

Ingress controllers are not services but are used for external traffic.

### Local Proxy for development

For troubleshooting `kubectl proxy` can be used. It provides access to internal services without permanently exposing them. It sets up a proxy server (typically on `127.0.0.1:8001`). You can then access objects via constructed URLs (`.../api/v1/namespaces/default/services/ghost:NAMED_PORT` for the `ghost` Service in ns `default`).

### How Services Communicate

The `kube-controller-manager` hosts two controllers critical for services: `Service Controller` and `Endpoint Controller`. Both controllers communicate to the API when Services/Endpoints are created/updated/removed. The API server then communicates with the cluster network plugin (Cilium, Calico, Flannel, ...) to ensure that routing and connectivity are applied on all nodes.

On each worker node networking agents (e.g. `cilium-node` for cilium and the `kube-proxy`), receive instructions and adapt the node networking accordingly. They do this by configuring the local firewall so that network traffic is correctly forwarded to the right pods. The `kube-proxy` mode is set when initializing the cluster by providing `--proxy-mode=iptables` or `--proxy-mode=ipvs`.

This means the service traffic flow is as follows:

1. External client request (optional)
2. Ingress routes traffic to a ClusterIp service (or NodePort or LoadBalancer)
3. `kube-proxy` managed firewall rules recognize traffic for a specific node port
4. The traffic is redirected from the NodePort to the ClusterIP of the service
5. The service consults its endpoints list (where Pod IPs running an application are tracked)
6. The request is forwarded to a Pod IP
7. If there multiple pods that match the service selector the traffic is distributed accross them

### Exposing an application

You can create a service for an existing Deployment, ReplicaSet, or Pod with (where PORT is the internal port of the deployment that should be used for traffic, e.g. 80 for an nginx deployments HTTP traffic):

`kubectl expose deployment/NAME --port=PORT --type=NodePort/ClusterIp`

A service configuration in a file includes three key settings:

1. `port` - The port exposed by the service for internal cluster access (e.g. 80)
2. `targetPort` The port on the pod where traffic is sent (defaults to `port` if unspecified)
3. `nodePort` - The randomly assigned port for external access in NodePort services

The range of ClusterIPs and NodePorts can be specified at API server startup.

Services are not limited to route traffic in the same namespace, they also can route to external resources or even legacy applications outside the cluster.

### DNS in Kubernetes

CoreDNS is the default DNS provider in Kubernetes and acts as a lightweight, extensible DNS server that runs within a Pod. It exposes a Service named `kube-dns` in the `kube-system` namespace. It manages DNS zones configured for the cluster to enable name resolution for a variety of resources.

It runs one or more servers that handle different DNS zones (e.g. `cluster.local` for the internal cluster domain). Each server can load one or more plugin chains to adapt how DNS queries are processed.

You can get the DNS IP by looking at `cat /etc/resolv.conf`. With that IP you can `dig @IP -x IP` to see that This DNS server is within the cluster. You can access pods from within another pod with e.g. `curl RESOURCE.NAMESPACE`

You can also inspect `kube-dns` further with `kubectl -n kube-system get svc kube-dns -o yaml`. There is also a configmap for dns viewable at `kubectl -n kube-system get configmaps coredns -oyaml`.

Process for adapting coredns config:

1. Backup current config: `kubectl -n kube-system get configmaps coredns -o yaml > coredns-backup.yaml`
2. Edit the dns configmap: `kubectl -n kube-system edit configmaps coredns`
3. Delete the coredns pods: `kubectl -n kube-system delete pod ...`

If you e.g. added a DNS rewrite in the config map, you would then be able to `dig REWRITTEN_URL` and see that it resolves to the service.

#### Verifying DNS registration

To confirm that DNS is working correctly inside the cluster you can run a simple test inside the cluster, e.g. from a Pod that has a shell and some simple network utilities installed. You can then use tools like `nslookup`, `dig`, `nc` or packet analyzers like `Wireshark` to debug.

In addition to general considerations on DNS you must also pay attention to the specific services in use and how it is applied to different Pods (e.g. matching labels to selector). You can also check `/etc/resolv.conf` inside a pod to verify that it points to the correct DNS server and checking Network Policies and firewalls that might be blocking DNS queries.

## Kubernetes Ingress

Builds upon services by creating a more efficient and flexible solution. It centralizes external access through a single entry point. You are able to route traffic based on request details like hostname, path, ... reducing the need for multiple load balancers. It is implemented with an Ingress Controller that operates independently from the `kube-controller-manager`.

Ingress Controllers are deployed as separate agents in the cluster. Ingress Rules (K8s resources) are used to determine how external traffic is routed to internal services. There are a few implementaions of Ingress Controllers like Traffic, Envoy, Countour, HAProxy. These can be deployed with manifest or Helm charts.

You can multiple Ingress controllers in a cluster (e.g. to separate configurations or purpose). Annotations are used to indicate which controller should process the rules. Make sure that only one controller handles certain traffic.

An example config for Ingress could look like this:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ghost
spec:
  rules:
  - host: ghost.192.168.99.100.nip.io
    http:
      paths:
      - path: /
        pathType: ImplementationSpecific
        backend:
          service
            name: ghost
            port:
              number: 2368
```

This routes root path traffic from `ghost.192.168.99.100.nip.io` to the ghost service on port `2368`. `ImplementationSpecific` Indicates that the path matching depends on the Ingress controller used.

You can manage Ingress through kubectl, e.g. to list `kubectl get ingress`, to edit `kubectl edit ingress NAME`, to delete `kubectl delete ingress NAME`.

Deploying an Ingress controller can be as simple as applying a pre-configured YAML. The full request flow would look like this:

1. External Request
2. Ingress Controller -> Matches host/path
3. Ingress Rule -> Routes to service
4. Service -> Selectors -> Pod
5. Pod

You can define multiple rules in an Ingress resource:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: multi-service-ingress
spec:
  rules:
  - host: ghost.192.168.99.100.nip.io
    http:
      paths:
      - path: /
        pathType: ImplementationSpecific
        backend:
          service:
            name: external
            port:
              number: 80
  - host: nginx.192.168.99.100.nip.io
    http:
      paths:
      - path: /
        pathType: ImplementationSpecific
        backend:
          service
            name: internal
            port:
              number: 8080
```

There are limitations to Ingress though:

- You can only route HTTP/HTTPS (no direct TCP, gRPC)
- Header-Based Routing is not possible out of the box
- Canary traffic splitting is not natively available

#### Configuring Ingress

### Gateway API to address Ingress shortcomings

Is comprised of four resource kinds to manage traffic in a cluster:

- `GatewayClass`: A group of gateways sharing configuration (think: blueprint for traffic handling)
- `Gateway`: Instance of that blueprint, listening to incoming requests
- `HTTPRoute`: Defines rules for routing HTTP traffic from a gateway to services (based on conditions like, hostname, path, header)
- `GRPCRoute`: Defines rules for routing gRPC traffic

#### GatewayClass

Scoped to the cluster as a whole. Defines a set of common properties and behaviors that any gateway references. This allows for implementing organizational policies, while application developers could define the actual routing rules somewhere else.

The configuration could look like this:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: example-class
spec:
  controllerName: example.com/gateway-controller
```

#### Gateway

Acts as the main entry point for external traffic entering the cluster. It specifies how requests should be received and processed by configuring one or more listeners. It allows behavior like "accept HTTP traffic on port 80" with configuration like this:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: example-gateway
spec:
  gatewayClassName: example-class
  listeners:
    - name: http
      protocol: HTTP
      port: 80
```

```
Gateways act as the "front door" of your Kubernetes cluster, while GatewayClasses determine who builds and manages that door. Developers define how traffic should flow, and administrators ensure that it is handled securely and consistently.
```

#### HTTPRoute

Defines how HTTP traffic is handled and routed within the Kubernetes cluster. It allows rules to be created based on hostnames, paths, headers, query params. It can enable weighted routing, canary deployments and more.

HTTPRoutes are associated with one or more gateways. This decouples it from the actual gateways and allows admins to handle gateway specific stuff, while devs can edit HTTPRoutes for application traffic. An example would be:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: example-httproute
spec:
  parentRefs:
    - name: example-gateway # The gateway it is associated with
  hostnames:
    - "www.example.com"
  rules:
    - matches:
        - path:
          type: PathPrefix
          value: /login
      backendRefs:
        - name: example-svc
          port: 8080
```

```
HTTPRoutes enable you to define how HTTP traffic reaches your applications, while Gateways control where traffic enters the cluster. Together, they provide a powerful and flexible model for managing application networking.
```

### Traffic Management with Service Meshes

Ingress might not be enough for service discovery, rate limiting, in-depth observability, ... Here are service mesh could help. It is an infrastructure layer that uses a network of proxies (typically as sidecars to application pods) to manage how services communicate with each other.

A service mesh operates on two planes: `Data Plane` and `Control Plane`, where the control plane oversees the proxies, provides their config, certificates, ... and more. The data plane comprises proxies that actually handle ingress/egress and mesh traffic between services.

Some examples of meshes include: Envoy, Istio, Linkerd.

#### Linkerd

Install the `linkerd` CLI as described on their site. You can run a pre-deploy check with `linkerd check --pre`. Afterwards enable the K8s gateway API `kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.2.1/standard-install.yaml` and install linkerd with `linkerd install --crds | kubectl apply -f -` and `linkerd install | kubectl apply -f -`. You can check the successful installation with `linkerd check`.

A `linkerd` GUI can be installed with `linkerd viz install | kubectl apply -f -` (check with `linkerd viz check`). This is by default only accessible on localhost. You would need to edit the service and deployment to allow access from a remote browser.

You can do this with: `kubectl -n linkerd-viz edit deploy web` and removing the `enforced-host`. Afterwards edit the service: `kubectl edit svc web -n linkerd-viz` and set a nodePort, so that you can access it from outside.

To add an object to the mesh, use `linkerd inject`: `kubectl -n accounting get deploy nginx-one -o yaml | linkerd inject - | kubectl apply -f -`. after that you should see that the namespace is meshed.
