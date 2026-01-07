# 05 – Cluster Initialization & Node Joining

**Goal:** Bootstrap the Kubernetes control plane, deploy the Pod network (Calico), and join worker nodes.
This document also explains how to **extend the cluster to a highly available control plane**.

---

## Part 1: Install Kubernetes Components

**Target:** Run on **ALL nodes**
(control-plane nodes and worker nodes)

We install and pin the following components:

* **kubeadm** – Cluster bootstrap and lifecycle tool
* **kubelet** – Node-level agent responsible for running pods
* **kubectl** – CLI for interacting with the cluster

---

### 1. Installing kubeadm, kubelet and kubectl


1. Update the `apt` package index and install packages needed to use the Kubernetes `apt` repository:
    
    ```shell
    sudo apt-get update
    # apt-transport-https may be a dummy package; if so, you can skip that package
    sudo apt-get install -y apt-transport-https ca-certificates curl gpg
    ```
    
2. Download the public signing key for the Kubernetes package repositories. The same signing key is used for all repositories so you can disregard the version in the URL:
    
    ```shell
    # If the directory `/etc/apt/keyrings` does not exist, it should be created before the curl command, read the note below.
    # sudo mkdir -p -m 755 /etc/apt/keyrings
    curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.35/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
    ```
    

#### Note:

In releases older than Debian 12 and Ubuntu 22.04, directory `/etc/apt/keyrings` does not exist by default, and it should be created before the curl command.

3. Add the appropriate Kubernetes `apt` repository. Please note that this repository have packages only for Kubernetes 1.35; for other Kubernetes minor versions, you need to change the Kubernetes minor version in the URL to match your desired minor version (you should also check that you are reading the documentation for the version of Kubernetes that you plan to install).
    
    ```shell
    # This overwrites any existing configuration in /etc/apt/sources.list.d/kubernetes.list
    echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.35/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
    ```
    
4. Update the `apt` package index, install kubelet, kubeadm and kubectl, and pin their version:
    
    ```shell
    sudo apt-get update
    sudo apt-get install -y kubelet kubeadm kubectl
    sudo apt-mark hold kubelet kubeadm kubectl
    ```
    
5. (Optional) Enable the kubelet service before running kubeadm:
    
    ```shell
    sudo systemctl enable --now kubelet
    ```
    
￼￼Shell￼
    sudo apt-get update
The kubelet is now restarting every few seconds, as it waits in a crashloop for kubeadm to tell it what to do

---

 ### One-Command Tools Setup (No Script Required)

To avoid maintaining scripts or managing permissions, the following **command group** can be executed directly on each node.

This approach:

- Runs in the **current shell**
    
- Requires **no file creation**
    
- Is ideal for manual provisioning and runbook-style execution
    

### Execute on each Kubernetes node

```
{
  sudo apt-get update
  # apt-transport-https may be a dummy package; if so, you can skip that package
  sudo apt-get install -y apt-transport-https ca-certificates curl gpg

  # If the directory `/etc/apt/keyrings` does not exist, it should be created before the curl command, read the note below.
  # sudo mkdir -p -m 755 /etc/apt/keyrings
  curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.35/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

  # This overwrites any existing configuration in /etc/apt/sources.list.d/kubernetes.list
  echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.35/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

  sudo apt-get update
  sudo apt-get install -y kubelet kubeadm kubectl
  sudo apt-mark hold kubelet kubeadm kubectl

  sudo systemctl enable --now kubelet
}


```


---

## Part 2: Initialize the Control Plane

**Target:** Run on **ONLY ONE initial control-plane node** (e.g. `cp-01`)

---

### 1. Bootstrap the Cluster

We initialize Kubernetes with flags that support **external load balancing** and **future HA expansion**.

```bash
sudo kubeadm init \
  --control-plane-endpoint "<LOAD_BALANCER_IP>:6443" \
  --upload-certs \
  --pod-network-cidr=192.168.0.0/16
```

**Why these flags matter**

* `--control-plane-endpoint`
  Points kubeconfig to HAProxy, not a single node
* `--upload-certs`
  Allows additional control-plane nodes to securely join later
* `--pod-network-cidr`
  Required by Calico

---

### 2. Configure kubectl Access

Run as a **non-root user**:

```bash
mkdir -p $HOME/.kube
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Alternatively (root only):

```bash
export KUBECONFIG=/etc/kubernetes/admin.conf
```

---

> **Note**:  Point Kube-VIP to the `admin.conf` 
> [Apply the Super-Admin Fix (Kube-VIP)](04b-lb-kube-vip.md#4-apply-the-super-admin-fix-critical-for-k8s-v129)

---

### 3. Save Join Commands

At the end of `kubeadm init`, **copy and store**:

* Worker join command
* Control-plane join command (includes `--control-plane` and cert key)

You will need both.

---

## Part 3: Install Pod Network (Calico)

**Target:** Run on the **control-plane node**

Until a CNI is installed:

* Nodes remain `NotReady`
* CoreDNS does not start

---

### Install Calico

```bash
curl https://raw.githubusercontent.com/projectcalico/calico/v3.31.3/manifests/calico-typha.yaml -o calico.yaml
kubectl apply -f calico.yaml
```

Verify:

```bash
kubectl get pods -n --all-namespaces
```

All pods must be in `Running` state.

---

## Part 4: Join Worker Nodes

**Target:** Run on **ALL worker nodes**

Paste the worker join command from `kubeadm init`:

```bash
sudo kubeadm join <LB_IP>:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>
```

If the token expired, generate a new one on a control-plane node:

```bash
kubeadm token create --print-join-command
```

---

## Part 5: High Availability – Adding More Control Plane Nodes

This section converts the cluster from **single control plane** to **HA**.

---

### 1. Prepare Additional Control Plane Nodes

On `cp-02`, `cp-03`:

* System hardening completed
* containerd installed and running
* Kubernetes tools installed

---

### 2. Join Additional Control Planes

Use the **control-plane join command** from `kubeadm init`.
It looks like this:

```bash
sudo kubeadm join <LB_IP>:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash> \
  --control-plane \
  --certificate-key <certificate-key>
```

This will:

* Download shared certificates
* Join etcd quorum
* Register the node as a control plane

---

### 3. Verify HA Control Plane

```bash
kubectl get nodes
```

Expected:

```text
cp-01   Ready   control-plane
cp-02   Ready   control-plane
cp-03   Ready   control-plane
worker-1 Ready  <none>
worker-2 Ready  <none>
```

---

## Part 6: Final Cluster Verification

Run from **Jump Host or Control Plane**:

```bash
kubectl get nodes
kubectl get pods -A
```

All nodes must be `Ready`.

---

## Optional: Allow Scheduling on Control Planes (Not Recommended for Prod)

```bash
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```

Remove LB exclusion label (if required):

```bash
kubectl label nodes --all node.kubernetes.io/exclude-from-external-load-balancers-
```

Refer [here](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/) for more

---
Next: [Storage Setup (OpenEBS)](06-storage-openebs.md)