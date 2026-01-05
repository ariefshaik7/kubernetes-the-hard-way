## 03 - Container Runtime Setup (containerd)

**Target:** Run on **ALL Kubernetes nodes**  
(control-plane nodes and worker nodes)

**Why:**  
DockerShim has been removed. Kubernetes communicates **directly** with containerd via CRI.

---

### 1. Install containerd

We use Docker’s official repository to install a **current, stable containerd build**.

```bash

# Add Docker's official GPG key:
sudo apt update
sudo apt install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Signed-By: /etc/apt/keyrings/docker.asc
EOF

sudo apt update

sudo apt install -y containerd.io
```

---

### 2. Configure containerd Cgroup Driver (**Critical**)

**Why this matters**

- Kubernetes uses **systemd** for cgroup management
    
- containerd defaults to **cgroupfs**
    
- A mismatch causes kubelet to **crash-loop**
    

We must explicitly enable `SystemdCgroup`.

```bash
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml > /dev/null

# Enable systemd cgroup driver
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

# Verify
grep SystemdCgroup /etc/containerd/config.toml
```

Expected output:

```text
SystemdCgroup = true
```

---

### 3. Restart containerd

```bash
sudo systemctl restart containerd
sudo systemctl enable containerd
```

---

## 4. One-Command containerd Setup (No Script Required)

To avoid maintaining scripts or managing permissions, the following **command group** can be executed directly on each node.

This approach:

- Runs in the **current shell**
    
- Requires **no file creation**
    
- Is ideal for manual provisioning and runbook-style execution
    

### Execute on each Kubernetes node

```bash
{
  echo ">>> Installing containerd dependencies"
  sudo apt update
  sudo apt install -y ca-certificates curl gnupg

  echo ">>> Adding Docker repository"
  sudo install -m 0755 -d /etc/apt/keyrings
  [ -f /etc/apt/keyrings/docker.gpg ] || \
    curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
    sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
  sudo chmod a+r /etc/apt/keyrings/docker.gpg

  echo ">>> Configuring Docker APT source"
  echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
https://download.docker.com/linux/ubuntu \
$(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

  echo ">>> Installing containerd"
  sudo apt update
  sudo apt install -y containerd.io

  echo ">>> Configuring systemd cgroup driver"
  sudo mkdir -p /etc/containerd
  containerd config default | sudo tee /etc/containerd/config.toml > /dev/null
  sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

  echo ">>> Restarting containerd"
  sudo systemctl restart containerd
  sudo systemctl enable containerd

  echo ">>> containerd installation complete"
}
```

---
Next: [Load Balancer ](04-load-balancer.md)