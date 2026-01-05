# 02 – Virtual Machine Preparation (System Hardening)

**Target:** Run on **ALL nodes**  
(control-plane nodes, worker nodes, load balancer nodes if applicable)

**Goal:** Prepare the Linux kernel, memory management, and networking stack to meet Kubernetes runtime requirements.

---

## 1. Disable Swap (**Mandatory**)

### Why this is required

Kubernetes relies on the **kubelet** for deterministic CPU and memory management.  
If swap is enabled, the Linux kernel may silently offload memory pages to disk, which:

- Breaks Kubernetes scheduling assumptions
    
- Causes unpredictable pod eviction behavior
    
- Leads to severe performance degradation
    

For this reason, Kubernetes **will not start** with swap enabled.

### Steps

```bash
# Disable swap immediately
sudo swapoff -a

# Disable swap permanently by commenting it out in /etc/fstab
sudo sed -i '/\sswap\s/s/^/#/' /etc/fstab

# Verify swap is disabled (output must be empty)
sudo swapon --show
```

---

## 2. Load Required Kernel Modules

### Why this is required

- **overlay**  
    Required by container runtimes (containerd) to support layered container filesystems.
    
- **br_netfilter**  
    Allows bridged traffic to be processed by `iptables`.  
    This is **mandatory** for:
    
    - Kubernetes Services
        
    - Network Policies
        
    - CNI plugins (Calico)
        

### Steps

```bash
# Configure modules to load on boot
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

# Load modules immediately (no reboot required)
sudo modprobe overlay
sudo modprobe br_netfilter
```

---

## 3. Kernel Network Configuration (Sysctl)

### Why this is required

Kubernetes nodes must behave like routers:

- Pods run on separate virtual networks
    
- Traffic must be forwarded between interfaces
    
- Packet filtering must occur at the bridge level
    

Without these settings:

- Pod-to-pod networking breaks
    
- Services do not route correctly
    
- Network policies fail silently
    

### Steps

```bash
# Configure Kubernetes-required sysctl parameters
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# Apply settings immediately (no reboot required)
sudo sysctl --system
```

---

## 4. Validation Checklist

After completing the above steps, validate the node state:

```bash
# Swap must be disabled
sudo swapon --show

# Required modules must be loaded
lsmod | grep -E 'overlay|br_netfilter'

# IP forwarding must be enabled
sysctl net.ipv4.ip_forward
```

All checks must return expected values before proceeding.

---

### 5. One-Command Host Preparation

To avoid distributing scripts or managing file permissions, the following **command group** can be executed directly on each node.

This approach:

- Requires **no file creation**
    
- Runs in the **current shell**
    
- Is ideal for manual provisioning and documentation-driven setups
    

### Execute on each node

```bash
{
  echo ">>> Disabling swap"
  sudo swapoff -a
  sudo sed -i '/\sswap\s/s/^/#/' /etc/fstab

  echo ">>> Configuring kernel modules"
  cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

  sudo modprobe overlay
  sudo modprobe br_netfilter

  echo ">>> Applying sysctl parameters"
  cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

  sudo sysctl --system

  echo ">>> Host preparation completed"
}
```

---

### 6. Post-Execution Validation

Run these checks **after** the command group completes:

```bash
# Swap must be disabled
sudo swapon --show

# Required kernel modules must be loaded
lsmod | grep -E 'overlay|br_netfilter'

# IP forwarding must be enabled
sysctl net.ipv4.ip_forward
```

All checks must return expected values before continuing.

---

Next: [Container Runtime Setup](03-container-runtime.md)