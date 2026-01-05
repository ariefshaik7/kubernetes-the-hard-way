# 06 – Storage Setup (OpenEBS)

**Goal:**  
Enable **dynamic Persistent Volume provisioning** so workloads (databases, monitoring, stateful services) can persist data.

**Why this matters:**  
Without a CSI provider, any `PersistentVolumeClaim` will remain in `Pending` state forever.

This cluster is **self-managed (bare metal / VM-based)**, so we must provide storage explicitly.

---

## 1. Storage Model Decision (Critical)

Before installing anything, it is important to understand **where data will live**.

### Local Storage (What we use)

In this setup, we use **OpenEBS LocalPV Hostpath**.

Characteristics:

- Volumes are backed by **node-local disk**
    
- Data is stored under a directory on the worker node
    
- No replication at the storage layer
    
- Near-zero overhead (no data-path sidecars)
    

Benefits:

- Extremely lightweight
    
- Ideal for:
    
    - VM-based clusters
        
    - Resource-constrained environments
        
    - Learning / platform engineering setups
        
- Matches real-world **on-prem bare-metal clusters**
    

Trade-offs:

- Volumes are **node-affined**
    
- If a node fails, the data does not move automatically
    
- Replication must be handled at the application level
    

---

### Replicated Storage (Not used here)

OpenEBS also provides **replicated block storage** (Mayastor):

- Data is synchronously replicated across nodes
    
- Requires:
    
    - NVMe disks
        
    - HugePages
        
    - Dedicated CPU and memory
        
- Storage engine runs in the data path
    

> ⚠️ **Not suitable for this cluster**
> 
> The nodes (`e2-small`, `e2-medium`) in our setup are resource-constrained.  
> Installing replicated storage engines **will cause memory pressure and instability**.

For this reason, **replicated storage is explicitly disabled**.

---

## 2. Prerequisites – Helm

**Target:** Run on the **Jump Host**  
(Any node with `kubectl` access)

OpenEBS is installed via **Helm** (Helm v3.2+ required).

### Install Helm (if not present)

```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-4
chmod 700 get_helm.sh
./get_helm.sh
```

Verify:

```bash
helm version
```

---

## 3. Install OpenEBS (LocalPV Only)

### Important Repository Change (v4+)

> ⚠️ **Note**
> 
> The old Helm repo  
> `https://openebs.github.io/charts`  
> is deprecated and used only for legacy OpenEBS (≤ v3.10).
> 
> **OpenEBS v4+ charts are hosted at:**
> 
> ```
> https://openebs.github.io/openebs
> ```

---

### Add the Correct Helm Repository

```bash
helm repo add openebs https://openebs.github.io/openebs
helm repo update
```

---

### Install OpenEBS (Replicated Storage Disabled)

This installs:

- LocalPV Hostpath
    
- LocalPV LVM
    
- LocalPV ZFS  
    …and **explicitly disables replicated storage (Mayastor)**.
    

```bash
helm install openebs openebs/openebs \
  --namespace openebs \
  --create-namespace \
  --set engines.replicated.mayastor.enabled=false
```

Why this is correct for your setup:

- Matches VM-based clusters
    
- Avoids etcd / NVMe / hugepages requirements
    
- Keeps control-plane and workers stable
    
---
### Install the "Lite" Version (for Local Setup)

We will disable the heavy "Replicated" engine and the unused "ZFS" and "LVM" engines to keep it lightweight (Hostpath only).

```
helm install openebs openebs/openebs \
  --namespace openebs \
  --create-namespace \
  --set engines.replicated.mayastor.enabled=false \
  --set engines.local.lvm.enabled=false \
  --set engines.local.zfs.enabled=false \
  --set engines.local.hostpath.enabled=true

```



---

## 4. Verify OpenEBS Installation

### Check Pods

```bash
kubectl get pods -n openebs
```

Expected (example – replicated storage disabled):

```text
openebs-localpv-provisioner-xxxxxx        1/1   Running
openebs-lvm-localpv-controller-xxxxxx    5/5   Running
openebs-lvm-localpv-node-xxxxxx           2/2   Running
openebs-zfs-localpv-controller-xxxxxx    5/5   Running
openebs-zfs-localpv-node-xxxxxx           2/2   Running
```

> You **should NOT** see:
> 
> - `openebs-io-engine`
>     
> - `openebs-etcd`
>     
> - `mayastor-*` pods
>     

That confirms replicated storage is disabled.

---

## 5. Verify StorageClasses

```bash
kubectl get storageclass
```

Expected output (example):

```text
NAME                 PROVISIONER        VOLUMEBINDINGMODE
openebs-hostpath     openebs.io/local   WaitForFirstConsumer
openebs-lvm          openebs.io/local   WaitForFirstConsumer
openebs-zfs          openebs.io/local   WaitForFirstConsumer
```

---

## 6. Set Default StorageClass

Kubernetes does not automatically choose a default storage class.

We explicitly set **`openebs-hostpath`** as default.

```bash
kubectl patch storageclass openebs-hostpath -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

Verify:

```bash
kubectl get sc
```

---

## 7. Verification – Smoke Test

### Create a Test PVC

**File:** `manifests/storage/test-pvc.yaml`

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: smoke-test-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

Apply:

```bash
kubectl apply -f manifests/storage/test-pvc.yaml
kubectl get pvc
```

Expected:

- `STATUS: Pending` (this is normal)
    

Reason:

- StorageClass uses `WaitForFirstConsumer`
    

---

### Bind the PVC with a Pod

```bash
kubectl run test-pod \
  --image=nginx \
  --restart=Never \
  --overrides='{
    "spec": {
      "volumes": [
        {"name": "vol", "persistentVolumeClaim": {"claimName": "smoke-test-pvc"}}
      ],
      "containers": [
        {
          "name": "nginx",
          "image": "nginx",
          "volumeMounts": [{"mountPath": "/data", "name": "vol"}]
        }
      ]
    }
  }'
```

Check:

```bash
kubectl get pvc
```

Expected:

- `STATUS: Bound`
    

---

### Cleanup

```bash
kubectl delete pod test-pod
kubectl delete pvc smoke-test-pvc
```

---

## 8. Production Notes (Important)

- LocalPV volumes are **node-local**
    
- No automatic failover
    
- Suitable for:
    
    - Dev / staging
        
    - On-prem clusters
        
    - Replicated applications (Prometheus, Elasticsearch, etc.)
        
- For strict data durability:
    
    - Use application-level replication
        
    - Or external distributed storage (Ceph, cloud block storage)
        

---

Next: [Encryption at Rest (Etcd)](07-etcd-encryption.md)