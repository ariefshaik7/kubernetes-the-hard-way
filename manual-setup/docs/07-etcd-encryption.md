# 07 – Encryption at Rest (Etcd)

**Goal:**  
Ensure Kubernetes Secrets (passwords, tokens, certificates) are **encrypted at rest** inside the Etcd datastore.

**Threat Model:**  
If an attacker gains access to:

- the node filesystem
    
- Etcd snapshots
    
- raw disks or backups
    

they **must not** be able to read secret data.

---

## Background

By default, Kubernetes stores Secrets as:

- **Base64-encoded**
    
- **Not encrypted**
    

Base64 is **encoding, not encryption**.

To fix this, we configure the Kubernetes API Server to encrypt Secrets **before writing them to Etcd**, using an **Encryption Provider Configuration**.

---

## 1. Generate an Encryption Key

**Target:** Run on **ONE control-plane node** (initial CP)

Generate a 32-byte random key and base64 encode it. You can use this command:

```shell
head -c 32 /dev/urandom | base64
```

⚠️ **Important**

- Store this key securely
    
- Losing it = losing access to encrypted secrets
    
- In production, this key should be backed up securely (Vault, HSM, offline storage)
    

---

## 2. Create Encryption Configuration

**Target:** Run on **ALL control-plane nodes**

### Write an encryption configuration file

#### Caution:

The encryption configuration file may contain keys that can decrypt content in etcd. If the configuration file contains any key material, you must properly restrict permissions on all your control plane hosts so only the user who runs the kube-apiserver can read this configuration.

### Create the Encryption Config File

```
mkdir -p /etc/kubernetes/enc
```

```bash
sudo tee /etc/kubernetes/enc/enc.yaml <<EOF
```yaml
---
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
      - configmaps
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: <BASE 64 ENCODED SECRET>
      - identity: {} 
EOF
```

```

### Why `identity` is included

```yaml
- identity: {}
```

This acts as a **fallback provider**:

- Allows reading unencrypted secrets
    
- Prevents cluster lockout during key rotation or migration
    
- Required for safe rollout
    

---

## 3. Configure the API Server

**Target:** Run on **ALL control-plane nodes**

Kubernetes control planes deployed via kubeadm run as **static pods**.  
We must modify the API Server manifest.

---

### Edit the API Server Manifest

```bash
sudo vim /etc/kubernetes/manifests/kube-apiserver.yaml
```

Locate the `command:` section and add the flag:

```yaml
---
#
# This is a fragment of a manifest for a static Pod.
# Check whether this is correct for your cluster and for your API server.
#
apiVersion: v1
kind: Pod
metadata:
  annotations:
    kubeadm.kubernetes.io/kube-apiserver.advertise-address.endpoint: 10.20.30.40:443
  creationTimestamp: null
  labels:
    app.kubernetes.io/component: kube-apiserver
    tier: control-plane
  name: kube-apiserver
  namespace: kube-system
spec:
  containers:
  - command:
    - kube-apiserver
    ...
    - --encryption-provider-config=/etc/kubernetes/enc/enc.yaml  # add this line
    volumeMounts:
    ...
    - name: enc                           # add this line
      mountPath: /etc/kubernetes/enc      # add this line
      readOnly: true                      # add this line
    ...
  volumes:
  ...
  - name: enc                             # add this line
    hostPath:                             # add this line
      path: /etc/kubernetes/enc           # add this line
      type: DirectoryOrCreate             # add this line
  ...
```

⚠️ **Critical Notes**

- YAML indentation must be exact
    
- A malformed file will crash the API Server
    
- Apply this change **on every control-plane node**
    
- Make sure that you use the **same** encryption configuration on each control plane host.
      

---

### What Happens Next

- Kubelet detects the manifest change
    
- API Server pod is restarted automatically
    
- Restart typically takes **30–60 seconds**
    

Verify:

```bash
kubectl get pods -n kube-system
```

Ensure all `kube-apiserver-*` pods return to `Running`.

---

## 4. Verification – Proving Encryption Works

We validate encryption by inspecting **raw Etcd data**.


This example shows how to check this for encrypting the Secret API.

1. Create a new Secret called `secret1` in the `default` namespace:
    
    ```shell
    kubectl create secret generic secret1 -n default --from-literal=mykey=mydata
    ```
    
    
2. Create a new Secret called `secret1` in the `default` namespace:
    
    ```shell
    sudo apt install etcd-client
    ```
    
    
3. Using the `etcdctl` command line tool, read that Secret out of etcd:
    
    ```shell
    sudo ETCDCTL_API=3 etcdctl \
       --cacert=/etc/kubernetes/pki/etcd/ca.crt   \
       --cert=/etc/kubernetes/pki/etcd/server.crt \
       --key=/etc/kubernetes/pki/etcd/server.key  \
       get /registry/secrets/default/secret1 | hexdump -C
    ```

    The output is similar to this (abbreviated):
    - ```hexdump
    00000000  2f 72 65 67 69 73 74 72  79 2f 73 65 63 72 65 74  |/registry/secret|
    00000010  73 2f 64 65 66 61 75 6c  74 2f 73 65 63 72 65 74  |s/default/secret|
    00000020  31 0a 6b 38 73 3a 65 6e  63 3a 61 65 73 63 62 63  |1.k8s:enc:aescbc|
    00000030  3a 76 31 3a 6b 65 79 31  3a c7 6c e7 d3 09 bc 06  |:v1:key1:.l.....|
    00000040  25 51 91 e4 e0 6c e5 b1  4d 7a 8b 3d b9 c2 7c 6e  |%Q...l..Mz.=..|n|
    00000050  b4 79 df 05 28 ae 0d 8e  5f 35 13 2c c0 18 99 3e  |.y..(..._5.,...>|
    [...]
    00000110  23 3a 0d fc 28 ca 48 2d  6b 2d 46 cc 72 0b 70 4c  |#:..(.H-k-F.r.pL|
    00000120  a5 fc 35 43 12 4e 60 ef  bf 6f fe cf df 0b ad 1f  |..5C.N`..o......|
    00000130  82 c4 88 53 02 da 3e 66  ff 0a                    |...S..>f..|
    0000013a
    ```
    
- Verify the stored Secret is prefixed with `k8s:enc:aescbc:v1:` which indicates the `aescbc` provider has encrypted the resulting data. Confirm that the key name shown in `etcd` matches the key name specified in the `EncryptionConfiguration` mentioned above. In this example, you can see that the encryption key named `key1` is used in `etcd` and in `EncryptionConfiguration`.
    
- Verify the Secret is correctly decrypted when retrieved via the API:
    
    ```shell
    kubectl get secret secret1 -n default -o yaml
    ```
    
    The output should contain `mykey: bXlkYXRh`, with contents of `mydata` encoded using base64.

### Decode the Secret

```shell
kubectl get secret secret1 -o jsonpath='{.data.mykey}' | base64 --decode
```

---

## 5. Encrypt Existing Secrets (Optional but Recommended)

Secrets created **before** enabling encryption remain unencrypted until rewritten.

To encrypt all existing secrets:

```bash
kubectl get secrets --all-namespaces -o json | kubectl replace -f -
```

This forces Kubernetes to:

- Read secrets
    
- Re-write them using the encryption provider
    

---
Next: [Service Load Balancing (MetalLB – Layer 4)](08-metallb.md)