	# 01-prerequisites

This document defines the **infrastructure, sizing, and architectural prerequisites** for building a **production-grade Kubernetes cluster**, along with a **reduced local footprint** for learning and experimentation.

The same architecture is used in both cases; **only scale and sizing differ**.

---

## 1. High-Level Architecture

Before provisioning, it is important to understand the target architecture.

We model a **real enterprise Kubernetes environment** with:

- Network isolation
    
- Bastion-based access
    
- Externalized control-plane access
    
- Horizontally scalable workers
    

### Architecture Characteristics

- **Infrastructure:** VM-based (cloud or on-prem)
    
- **Access Model:** Bastion (Jump Host)
    
- **Control Plane:** Highly Available (3 nodes)
    
- **Workers:** Horizontally scalable
    
- **Load Balancing:** External L4 for API + Services
    
- **Storage:** CSI-based dynamic provisioning
    

---

## 2. Production Infrastructure Sizing (Reference Architecture)

This section defines the **baseline production topology**.  
Actual sizing **must always be adjusted** based on:

- Workload characteristics
    
- Request/limit strategy
    
- Traffic volume
    
- SLO/SLA requirements
    

### Production VM Topology

| Role                    | Hostnames                 | GCP Machine Type | Description                                                  |
| ----------------------- | ------------------------- | ---------------- | ------------------------------------------------------------ |
| **Bastion / Jump Host** | `bastion-host`            | `e2-micro`       | Secure administrative entry point. No workloads run here.    |
| **Load Balancer**       | `lb-01`                   | `e2-small`       | Runs HAProxy for Kubernetes API traffic. Public entry point. |
| **Control Plane**       | `cp-01`, `cp-02`, `cp-03` | `e2-medium`      | Runs API Server, Scheduler, Controller Manager, Etcd.        |
| **Worker Nodes**        | `worker-01`, `worker-02`  | `e2-medium`      | Executes application workloads and platform services.        |

### Production Notes

- **Three control planes** are mandatory for quorum and fault tolerance.
    
- **No workloads** should run on control plane nodes.
    
- Worker count should scale independently from control planes.
    
- Instance types and node counts **must be revisited** as load increases.
    

> ⚠️ This sizing represents a **minimum viable production baseline**, not a universal recommendation.

---

## 3. Local / Bare-Metal Replication (Reduced Footprint)

For **local development, learning, and documentation**, the same architecture can be replicated with **fewer nodes and reduced resources**.

### Local VM Topology (Minimal)

|Role|Hostname|Suggested Specs|Purpose|
|---|---|---|---|
|**Jump Host**|`jump-host`|512 MB RAM, 1 vCPU|Optional. Can be replaced by host machine.|
|**Load Balancer**|`lb`|512 MB RAM, 1 vCPU|HAProxy for API traffic.|
|**Control Plane**|`control-plane`|2 GB RAM, 2 vCPU|Single-node control plane for learning.|
|**Worker Node**|`worker-1`|2 GB RAM, 1 vCPU|Runs workloads and platform components.|

### Local Constraints

- Single control plane **is not HA**.
    
- Resource pressure may occur during upgrades or add-ons.
    
- Suitable for:
    
    - kubeadm internals
        
    - networking concepts
        
    - storage testing
        
    - automation pipelines
        

---

## 4. Physical Host Requirements (Local Setup)

Applies **only** when running locally using Multipass / KVM / VirtualBox.

- **OS:** Linux, macOS, or Windows (WSL2)
    
- **CPU:** 4 cores minimum
    
- **RAM:** 8 GB minimum (16 GB recommended)
    
- **Virtualization:** Multipass / KVM / VirtualBox / Vagrant
    

---

## 5. Software Stack (Bill of Materials)

This cluster is designed as a **platform**, not just a Kubernetes install.

### Cluster Fundamentals

- **OS:** Ubuntu 22.04 LTS / 24.04 LTS
    
- **Container Runtime:** `containerd`
    
- **Bootstrap Tool:** `kubeadm`
    

### Networking & Traffic

- **CNI:** Calico  
    _Pod networking + Network Policies_
    
- **Control Plane Load Balancer:** HAProxy  
    _External API endpoint_
    
- **Service Load Balancer:** MetalLB (L2 mode)  
    _Bare-metal style LoadBalancer services_
    
- **L7 Gateway:** Envoy Gateway  
    _Gateway API–based traffic management_


> ⚠️  **MetalLB Mode Selection:**  
L2 mode is used here for **simplicity and local/on-prem demonstrations**.  
In real production environments, especially at scale, **BGP mode** is preferred as it enables:
BGP mode requires network support and coordination with the underlying routing infrastructure.

### Storage & Security

- **CSI:** OpenEBS (LocalPV Hostpath)  
    _Dynamic persistent volumes_
    
- **Secrets Encryption:** Etcd Encryption at Rest  
    _Protects Kubernetes Secrets_
    

---

## 6. Network Planning (Example – Local)

Example subnet used for local replication. Adjust as required.

- **Subnet:** `10.47.160.0/24`
    
- **Gateway:** `10.47.160.1`
    
- **Jump Host:** `10.47.160.10`
    
- **Control Plane:** `10.47.160.11`
    
- **Worker Node:** `10.47.160.12`
    
- **MetalLB Pool:** `10.47.160.200 – 10.47.160.210`
    

---
Next: [Node Hardening](02-node-hardening.md)
