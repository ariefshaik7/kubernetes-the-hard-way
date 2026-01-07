# 04 – Load Balancer & High Availability

This section establishes the **"Front Door"** of your Kubernetes cluster.

In a production cluster, you cannot have your workers talk directly to a single Control Plane node. If that node dies, your cluster becomes unmanageable. Instead, we use a **Load Balancer** to distribute traffic across multiple Control Plane nodes.

![HA Kubernetes Architecture](manual-setup/architecture/lb-architecture.png)

## The Two Architectures

This section covers **API load balancing only**.
It does **not** handle Kubernetes `Service.type=LoadBalancer`, which is addressed later using **MetalLB**.

There are two industry-standard ways to achieve this. **Choose ONE path** based on your environment.

### Path A: External Load Balancer (Classic Enterprise)

**Uses:** HAProxy + Keepalived (on separate VMs)

- **How it works:** You create two dedicated VMs (`lb-01`, `lb-02`). They run **Keepalived** to pass a Virtual IP (VIP) back and forth. **HAProxy** runs on them to balance traffic.
    
- **Pros:** Extremely robust. Decouples the "Networking" layer from the "Kubernetes" layer. Easier to debug networking issues.
    
- **Cons:** Requires 2 extra VMs.
    
- **Best For:** On-Premise Data Centers, traditional VM environments, and **Learning Labs** (easier to visualize).
    

Next: ** [04a-haproxy-keepalived.md](04a-lb-haproxy.md)

---

### Path B: Kube-VIP (Modern Bare-Metal)

**Uses:** Kube-VIP (Static Pods)

- **How it works:** The Load Balancer runs **inside** the Control Plane nodes themselves. There are no extra VMs. The nodes talk to each other to decide who holds the VIP.
    
- **Pros:** Zero extra hardware/VMs required. Simplified architecture.
    
- **Cons:** Slightly harder to debug if the cluster itself is failing to bootstrap.
    
- **Best For:** Bare Metal servers, Edge devices, and **Homelabs** where RAM is scarce.
    

Next: **[04b-kube-vip.md](04b-lb-kube-vip.md)**

---
