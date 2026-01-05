# 04(a) – Load Balancer (HAProxy/Keepalived)

This section installs the **front door** of the cluster (HAProxy) and the **execution engine** (containerd).

---

## Part 1: HAProxy Load Balancer Setup

**Target:** Run this **ONLY** on the **Load Balancer node**  
**Goal:** Expose a **stable Kubernetes API endpoint** that forwards traffic to the control plane.  
This allows future control-plane scaling **without changing kubeconfig**.

---

### 1. Install HAProxy

```bash
sudo apt update
sudo apt install -y haproxy
```

---

### 2. Configure HAProxy for Kubernetes API

HAProxy will:

- Listen on port `6443`
    
- Forward TCP traffic to control-plane nodes
    
- Perform basic TCP health checks
    

**Edit the configuration file:**

```bash
sudo vim /etc/haproxy/haproxy.cfg
```

**Append the following block at the end of the file**  
(Replace `10.47.160.11` with your actual control-plane IP)

```haproxy
# ---------------------------------------------------------------------
# Kubernetes API Server Load Balancer
# ---------------------------------------------------------------------
frontend kubernetes-frontend
    bind *:6443
    mode tcp
    option tcplog
    default_backend kubernetes-backend

backend kubernetes-backend
    mode tcp
    balance roundrobin
    option tcp-check

    # Format: server <name> <IP>:6443 check
    server cp-01 10.47.160.11:6443 check fall 3 rise 2

    # Future HA scaling
    # server cp-02 10.47.160.12:6443 check fall 3 rise 2
    # server cp-03 10.47.160.13:6443 check fall 3 rise 2
```

---

### 3. Restart & Verify HAProxy

```bash
sudo systemctl restart haproxy
sudo systemctl enable haproxy

# Verify status
sudo systemctl status haproxy
```

**Verification**

```bash
nc -vz <LB-IP> 6443
```

A successful TCP connection confirms the frontend is listening correctly.

---
Next: [Cluster Initialization & Node Joining](05-k8s-bootstrap.md)