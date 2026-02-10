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
-
