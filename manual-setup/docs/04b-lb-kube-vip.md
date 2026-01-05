# 04(b) - Load Balancer (Kube-VIP)

This section installs **Kube-VIP**, a high-availability Virtual IP manager that runs directly on your Control Plane nodes. This removes the need for a dedicated external Load Balancer VM.

---

## Part 1: Kube-VIP Setup

**Target:** Run this on **Control Plane Node 1 (`cp-1`)** initially.

**Goal:** Generate a "Static Pod Manifest" that forces the Kubelet to start the Load Balancer before the cluster even initializes.

---

### 1. Prerequisites

Ensure you are logged into `cp-1` as **root** (`sudo -i`).

A. Create the Manifest Directory

Kubeadm expects static pods to live here.

Bash

```
mkdir -p /etc/kubernetes/manifests/
```

B. Determine your Network Interface

You must bind the VIP to the correct physical cable.

Bash

```
ip a
```

- Look for your main IP (e.g., `10.47.160.xx`).
    
- Note the interface name (usually `eth0`, `ens3`, or `enp0s3`).
    

---

### 2. Configure Variables

Set these environment variables. **Customize the VIP and Interface** for your network.

Bash

```
# 1. The Virtual IP you want to assign to the cluster (Must be unused!)
export VIP=10.47.160.50

# 2. Your Physical Interface Name (Found in Step 1)
export INTERFACE=ens3

# 3. Get the latest Kube-VIP version automatically
export KVVERSION=$(curl -sL https://api.github.com/repos/kube-vip/kube-vip/releases | jq -r ".[0].name")
```

---

### 3. Generate the Manifest

We use `ctr` (the Containerd CLI) to run the image strictly to generate the configuration YAML.

Bash

```
# 1. Pull the image manually to the default namespace (Avoids startup delays)
ctr image pull ghcr.io/kube-vip/kube-vip:$KVVERSION

# 2. Create an alias to run the generator
alias kube-vip="ctr run --rm --net-host ghcr.io/kube-vip/kube-vip:$KVVERSION vip /kube-vip"

# 3. Generate the YAML file
kube-vip manifest pod \
    --interface $INTERFACE \
    --vip $VIP \
    --controlplane \
    --arp \
    --leaderElection | tee /etc/kubernetes/manifests/kube-vip.yaml
```

---

### 4. Apply the "Super-Admin" Fix (Critical for K8s v1.29+)

Kubernetes v1.29 changed how permissions work during bootstrap. We must point Kube-VIP to the special `super-admin.conf` file, or the cluster initialization will time out.

Bash

```
sed -i 's#path: /etc/kubernetes/admin.conf#path: /etc/kubernetes/super-admin.conf#' /etc/kubernetes/manifests/kube-vip.yaml
```

**Verify the File:**

Bash

```
cat /etc/kubernetes/manifests/kube-vip.yaml
```

- Check that `value: 10.47.160.50` (Your VIP) is correct.
    
- Check that `path: /etc/kubernetes/super-admin.conf` exists near the bottom.
    

---

### 5. Pre-Pull the Image for Kubelet

To ensure the fastest possible startup during the next step (Initialization), we pull the image into the k8s namespace `crictl`.

Bash

```
crictl pull ghcr.io/kube-vip/kube-vip:$KVVERSION
```

---

What Happens Next?

You will not see the Pod running yet. It will only start when you run kubeadm init in the next step.

Next: [Cluster Initialization & Node Joining](05-k8s-bootstrap.md)