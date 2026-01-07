# 01 – Prerequisites

This document defines the **infrastructure, sizing, and architectural prerequisites** for building a **production reference Kubernetes cluster**, along with a **reduced local footprint** for learning and experimentation.

The **architecture remains the same** in both cases; **only scale and sizing differ**.

---

## 1. High-Level Architecture

Before provisioning any infrastructure, it is important to understand the **target architecture and design assumptions**.

This cluster models a **real-world enterprise Kubernetes environment**, emphasizing:

- Network isolation
- Bastion-based administrative access
- Externalized control plane access
- Horizontally scalable worker nodes

### Architecture Characteristics

- **Infrastructure:** VM-based 
- **Access Model:** Bastion / Jump Host
- **Control Plane:** Highly Available (3 nodes, stacked etcd)
- **Workers:** Horizontally scalable
- **API Load Balancing:** External L4 (HAProxy or Kube-VIP)
- **Service Load Balancing:** MetalLB (L2 mode)
- **Storage:** CSI-based dynamic volume provisioning

---

## 2. Production Infrastructure Sizing (Reference Architecture)

This section defines a **baseline production reference topology**.

Actual sizing **must always be adjusted** based on:

- Workload characteristics
- Pod resource requests and limits
- Traffic patterns
- Availability and performance requirements (SLOs / SLAs)

### Production VM Topology

| Role                    | Hostnames                 | Suggested Specs            | Description                                                  |
| ----------------------- | ------------------------- | -------------------------- | ------------------------------------------------------------ |
| **Bastion / Jump Host** | `bastion-host`            | 512 MB RAM, 1 vCPU         | Secure administrative entry point. No workloads run here.    |
| **Load Balancer**       | `lb-01`                   | 512 MB RAM, 1 vCPU         | External API endpoint (HAProxy / equivalent).                |
| **Control Plane**       | `cp-01`, `cp-02`, `cp-03` | ≥ 2 GB RAM, 2 vCPU each    | Runs API Server, Scheduler, Controller Manager, and Etcd.    |
| **Worker Nodes**        | `worker-01`, `worker-02`  | ≥ 2 GB RAM, 1–2 vCPU each  | Runs application workloads and platform services.            |

### Production Notes

- **Three control plane nodes** are required for etcd quorum and fault tolerance.
- **No workloads** should run on control plane nodes.
- Worker nodes scale **independently** from the control plane.
- Instance sizes and counts **must be revisited** as cluster load increases.

> ⚠️ This sizing represents a **minimum viable production reference**, not a universal recommendation.

---

## 3. Local / Bare-Metal Replication (Reduced Footprint)

For **local development, learning, and documentation**, the same architecture can be replicated with **fewer nodes and reduced resources**.

### Local VM Topology (Minimal)

| Role              | Hostname        | Suggested Specs    | Purpose                                    |
| ----------------- | --------------- | ------------------ | ------------------------------------------ |
| **Jump Host**     | `jump-host`     | 512 MB RAM, 1 vCPU | Optional. Can be replaced by host machine. |
| **Load Balancer** | `lb`            | 512 MB RAM, 1 vCPU | API traffic load balancing.                |
| **Control Plane** | `control-plane` | 2 GB RAM, 2 vCPU   | Single-node control plane for learning.    |
| **Worker Node**   | `worker-1`      | 2 GB RAM, 1 vCPU   | Runs workloads and platform components.    |

### Local Constraints

- Single control plane **is not highly available**.
- Resource pressure may occur during upgrades or add-on installation.
- Suitable for:
  - kubeadm internals
  - networking concepts
  - storage testing
  - automation and CI experimentation

---

## 4. Physical Host Requirements (Local Setup)

Applies **only** when running locally using KVM or similar hypervisors.

- **OS:** Linux or macOS
- **CPU:** 4 cores minimum
- **RAM:** 8 GB minimum (16 GB recommended)
- **Virtualization:** KVM / Multipass / Vagrant

---

## 5. Software Stack (Bill of Materials)

This cluster is designed as a **platform**, not just a Kubernetes installation.

### Cluster Fundamentals

- **OS:** Ubuntu 22.04 LTS / 24.04 LTS
- **Container Runtime:** `containerd`
- **Bootstrap Tool:** `kubeadm`

### Networking & Traffic

- **CNI:** Calico  
  _Pod networking and Network Policies_

- **Control Plane Load Balancer:** HAProxy  
  _External Kubernetes API endpoint_

- **Service Load Balancer:** MetalLB (L2 mode)  
  _Bare-metal style `LoadBalancer` Services_

- **L7 Gateway:** Envoy Gateway  
  _Gateway API–based HTTP routing_

> ⚠️ **MetalLB Mode Selection**  
> L2 mode is used here for **simplicity and on-prem / lab environments**.  
> In larger production environments, **BGP mode** is often preferred but requires routing infrastructure support.

### Storage & Security

- **CSI:** OpenEBS (LocalPV Hostpath)  
  _Dynamic persistent volumes_

- **Secrets Encryption:** Etcd encryption at rest  
  _Protects Kubernetes Secrets_

---

Next: [Node Hardening](02-node-hardening.md)
