# 08 – Service Load Balancing (MetalLB – Layer 4)

**Goal:**  
Provide Kubernetes `Service.type=LoadBalancer` with a **real, routable IP address** in bare-metal or self-managed VM environments.

---

## Problem Statement

In cloud environments (AWS, GCP, Azure), `Service: LoadBalancer` is backed by a managed load balancer.

In **bare-metal or self-managed clusters**:

- No external load balancer exists
    
- Services remain stuck in `Pending`
    
- Applications are unreachable from outside the cluster
    

---

## Solution: MetalLB (Layer 4)

**MetalLB** is a Kubernetes-native load balancer implementation for bare-metal clusters.

- Assigns IPs from your **local network**
    
- Uses standard **ARP** (Layer 2)
    
- Integrates directly with Kubernetes Services
    
- Cloud-provider agnostic
    

This setup uses **L2 mode**, which is the simplest and most reliable for homelabs and on-prem clusters.

---

## Part 1: Network Planning (Finding Free IP Addresses)

**Target:** Run on **Jump Host or Laptop**

### Why this matters

MetalLB does **not** create IPs , it **claims existing ones**.  
Using an IP already assigned by DHCP or another host causes:

- IP conflicts
    
- Packet loss
    
- Intermittent outages
    

You must **prove** the IP range is unused.

---

### 1. Identify Your Subnet

```bash
ip a        # Linux
ifconfig   # macOS
```

Example output:

```
inet 10.47.160.57/24
```

This means your network is:

```
10.47.160.0/24
```

---

### 2. Scan Active Hosts (Scientific Method)

Install `nmap` if required:

```bash
sudo apt install -y nmap
```

Perform a ping scan:

```bash
nmap -sn 10.47.160.0/24
```

This lists **all IPs currently in use**.

---

### 3. Choose a Safe Range

Typical findings:

- `.1` → Router
    
- `.10–.100` → DHCP pool
    
- `.200+` → Usually unused
    

**Recommended approach:**

- Reserve a contiguous block (e.g. 10 IPs)
    
- Example:
    
    ```
    10.47.160.200 – 10.47.160.210
    ```
    

---

### 4. Verify with Ping

Before committing, verify at least one IP:

```bash
ping -c 3 10.47.160.200
```

Interpretation:

- `Destination Host Unreachable` →  Free
    
- `Reply from …` → ❌ In use
    

Repeat until confident.

---

## Part 2: Enable Strict ARP (Required)

**Target:** Run on **Jump Host**

### Why this is required

MetalLB in L2 mode relies on ARP announcements.

By default:

- `kube-proxy` may respond incorrectly
    
- Traffic can be blackholed
    

We must enable **strict ARP**.

---

### Apply the Patch

```bash
# see what changes would be made, returns nonzero returncode if different
kubectl get configmap kube-proxy -n kube-system -o yaml | \
sed -e "s/strictARP: false/strictARP: true/" | \
kubectl diff -f - -n kube-system

# actually apply the changes, returns nonzero returncode on errors only
kubectl get configmap kube-proxy -n kube-system -o yaml | \
sed -e "s/strictARP: false/strictARP: true/" | \
kubectl apply -f - -n kube-system
```

Verify:

```bash
kubectl get configmap kube-proxy -n kube-system -o yaml | grep strictARP
```

---

## Part 3: Install MetalLB

**Target:** Run on **Jump Host**

### Install Manifests

```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.15.3/config/manifests/metallb-native.yaml
```

Wait for pods:

```bash
kubectl get pods -n metallb-system
```

All pods must be `Running`.

This will deploy MetalLB to your cluster, under the `metallb-system` namespace. The components in the manifest are:

- The `metallb-system/controller` deployment. This is the cluster-wide controller that handles IP address assignments.
- The `metallb-system/speaker` daemonset. This is the component that speaks the protocol(s) of your choice to make the services reachable.
- Service accounts for the controller and speaker, along with the RBAC permissions that the components need to function.
The installation manifest does not include a configuration file. MetalLB’s components will still start, but will remain idle until you start deploying resources.
---

## Part 4: Configure IP Address Pool

MetalLB requires **explicit IP ownership configuration**.

### Create IP Pool Manifest

**File:** `metallb-pool.yaml`

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: production-pool
  namespace: metallb-system
spec:
  addresses:
    - 10.47.160.200-10.47.160.210
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: production-advertisement
  namespace: metallb-system
```

Apply:

```bash
kubectl apply -f metallb-pool.yaml
```

---

## Part 5: Verification (Smoke Test)

### Create a Test Service

```bash
kubectl create deployment lb-test --image=nginx --replicas=1
kubectl expose deployment lb-test --type=LoadBalancer --port=80
```

Check service:

```bash
kubectl get svc lb-test
```

Expected:

- `EXTERNAL-IP` is **assigned**
    
- IP falls within your MetalLB pool
    

Example:

```
lb-test   LoadBalancer   10.96.12.34   10.47.160.200
```

---

## Operational Notes (Production Reality)

- MetalLB **does not persist data**
    
- IPs are reclaimed if services are deleted
    
- L2 mode works best on flat networks
    
- For multi-subnet or routed environments → BGP mode is preferred
    


---
Next: [07 – Application Routing (Envoy Gateway – Layer 7)](09-gateway-api.md)