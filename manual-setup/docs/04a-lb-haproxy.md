# 04(a) – Load Balancer (HAProxy + Keepalived)

This section sets up a **High Availability (HA)** control plane entry point using dedicated Load Balancer nodes.

We use **Keepalived** to manage a floating **Virtual IP (VIP)** and **HAProxy** to load balance traffic to the API Servers. This ensures that even if one node fails, the Kubernetes API remains accessible at the same IP address.

## Architecture: Dedicated VMs (OS Services)

- **Target Nodes:** `lb-01`, `lb-02` (Separate from the Kubernetes Cluster)
    
- **VIP:** `10.47.160.50` (Example)
    
- **Method:** Systemd services (`apt install haproxy keepalived`)
    

---

## Part 1: Prerequisites

**Target:** Run on **BOTH** `lb-01` and `lb-02`.

### 1. Install Software

```
sudo apt update
sudo apt install -y haproxy keepalived
```

### 2. Configure Non-Local Bind

This allows the backup load balancer to "listen" on the VIP even though it doesn't own it yet.

```
echo "net.ipv4.ip_nonlocal_bind=1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

---

## Part 2: Configure HAProxy

Target: Run on BOTH lb-01 and lb-02.

Goal: Distribute traffic from port 6443 to all Control Plane nodes.

1. **Edit the config:**
    
    
    ```
    sudo vim /etc/haproxy/haproxy.cfg
    ```
    
2. Append this block to the end:
    
    (Replace IPs with your actual Control Plane IPs)
    
    ```
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
        option tcp-check
        balance roundrobin
    
        # Format: server <name> <IP>:6443 check
        server cp-01 <Control-plane-1><IP>:6443 check fall 3 rise 2
        server cp-02 <Control-plane-2><IP>:6443 check fall 3 rise 2
        server cp-03 <Control-plane-3><IP>:6443 check fall 3 rise 2
    ```
    

---

## Part 3: Configure Keepalived Health Check Script

**Target:** Run on **BOTH** `lb-01` and `lb-02`.

Instead of just checking if the process is running, we will use a script that verifies the Load Balancer is actually responding on port `6443`.

1. **Create the script:**
    
    Bash
    
    ```
    sudo vim /etc/keepalived/check_apiserver.sh
    ```
    
2. **Paste the content:**
    
    Bash
    
    ```
    #!/bin/sh
    
    errorExit() {
        echo "*** $*" 1>&2
        exit 1
    }
    
    # Check if we can talk to localhost:6443 (HAProxy Front End)
    # The --max-time 2 ensures we fail fast if HAProxy is hung
    curl -sfk --max-time 2 https://localhost:6443/healthz -o /dev/null || errorExit "Error GET https://localhost:6443/healthz"
    ```
    
3. **Make it executable:**
    
    Bash
    
    ```
    sudo chmod +x /etc/keepalived/check_apiserver.sh
    ```
    

---

## Part 4: Configure Keepalived

**Target:** Run specific configs on Master and Backup nodes.

### On Master Node (`lb-01`)

**File:** `/etc/keepalived/keepalived.conf`

Code snippet

```
# Global Definitions
global_defs {
    router_id LVS_DEVEL
}

# Define the script we created in Part 3
vrrp_script check_apiserver {
  script "/etc/keepalived/check_apiserver.sh"
  interval 3
  weight -2
  fall 10
  rise 2
}

# The VRRP Instance
vrrp_instance VI_1 {
    state MASTER             # <--- MASTER on lb-01
    interface ens3           # <--- CHECK YOUR INTERFACE (ip a)
    virtual_router_id 51     # Must match on all nodes
    priority 101             # <--- 101 on Master
    authentication {
        auth_type PASS
        auth_pass 42
    }
    virtual_ipaddress {
        10.47.160.50/24      # <--- The VIP
    }
    track_script {
        check_apiserver
    }
}
```

### On Backup Node (`lb-02`)

**File:** `/etc/keepalived/keepalived.conf`

Code snippet

```
# Global Definitions
global_defs {
    router_id LVS_DEVEL
}

vrrp_script check_apiserver {
  script "/etc/keepalived/check_apiserver.sh"
  interval 3
  weight -2
  fall 10
  rise 2
}

vrrp_instance VI_1 {
    state BACKUP             # <--- BACKUP on lb-02
    interface ens3           # <--- CHECK YOUR INTERFACE
    virtual_router_id 51
    priority 100             # <--- 100 on Backup
    authentication {
        auth_type PASS
        auth_pass 42
    }
    virtual_ipaddress {
        10.47.160.50/24      # <--- The VIP
    }
    track_script {
        check_apiserver
    }
}
```

---

## Part 5: Start & Verify

1. **Start Services (Both Nodes):**
    
    Bash
    
    ```
    sudo systemctl restart haproxy keepalived
    sudo systemctl enable haproxy keepalived
    ```
    
2. **Verify VIP (On Master):**
    
    Bash
    
    ```
    ip a | grep 10.47.160.50
    ```
    
    _Should return the IP._
    
3. **Verify Failover:**
    
    - Stop Keepalived on Master: `sudo systemctl stop keepalived`
        
    - Check Backup: `ip a | grep 10.47.160.50` (Should see IP).
        
4. **Verify API Access:**
    
    Bash
    
    ```
    # Run from your laptop or jump host
    nc -v 10.47.160.50 6443
    ```
    

Refer [here](https://github.com/kubernetes/kubeadm/blob/main/docs/ha-considerations.md) for more

---

Next: [Cluster Initialization & Node Joining](05-k8s-bootstrap.md)