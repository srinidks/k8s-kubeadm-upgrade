# k8s-kubeadm-upgrade
# Kubernetes Cluster Upgrade Guide: v1.29 → v1.30

A step-by-step guide for upgrading a kubeadm-based Kubernetes HA cluster (3 control planes + 2 worker nodes) from v1.29 to v1.30, including etcd backup and verification.

---

## Cluster Overview

| Node | Role | Version Before | Version After |
|------|------|----------------|---------------|
| masternode1 | control-plane | v1.29.13 | v1.30.x |
| masternode2 | control-plane | v1.29.14 | v1.30.x |
| masternode3 | control-plane | v1.29.14 | v1.30.x |
| workernode1 | worker | v1.29.13 | v1.30.x |
| workernode2 | worker | v1.29.13 | v1.30.x |

---

## ⚠️ Important Rules

- **Never skip minor versions.** You must go `1.29 → 1.29.15 → 1.30.x` — not directly to 1.30.
- Always upgrade **control plane nodes first**, then worker nodes.
- Always run `kubeadm upgrade apply` **only on the first control plane** (masternode1). Other control planes use `kubeadm upgrade node`.
- `kubeadm upgrade apply` does **not** upgrade `kubelet` or `kubectl` — those must be upgraded manually on each node.
- Take a **backup before every upgrade phase**.

---

## Upgrade Order

```
BACKUP
  ↓
masternode1  →  upgrade apply  →  drain  →  kubelet/kubectl  →  uncordon
  ↓
masternode2  →  upgrade node   →  drain  →  kubelet/kubectl  →  uncordon
  ↓
masternode3  →  upgrade node   →  drain  →  kubelet/kubectl  →  uncordon
  ↓
workernode1  →  upgrade node   →  drain  →  kubelet/kubectl  →  uncordon
  ↓
workernode2  →  upgrade node   →  drain  →  kubelet/kubectl  →  uncordon
  ↓
(Repeat for next minor version hop)
```

---

## PHASE 0 — Backup (Run on masternode1)

### Install etcdctl matching your etcd version (3.5.x)

```bash
ETCD_VER=v3.5.16
curl -L https://github.com/etcd-io/etcd/releases/download/${ETCD_VER}/etcd-${ETCD_VER}-linux-amd64.tar.gz \
  -o /tmp/etcd.tar.gz

tar xzf /tmp/etcd.tar.gz -C /tmp/
sudo mv /tmp/etcd-${ETCD_VER}-linux-amd64/etcdctl /usr/local/bin/etcdctl3

# Verify
etcdctl3 version
```

### Take etcd snapshot

```bash
ETCDCTL_API=3 etcdctl3 snapshot save /root/etcd-backup-$(date +%Y%m%d-%H%M%S).db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

### Verify the snapshot

```bash
ETCDCTL_API=3 etcdctl3 snapshot status /root/etcd-backup-*.db --write-out=table
```

### Backup PKI certs and configs

```bash
sudo cp -r /etc/kubernetes /root/k8s-backup-$(date +%Y%m%d)
sudo cp -r /var/lib/etcd /root/etcd-data-backup-$(date +%Y%m%d)
```

### Copy backups off the cluster (from your local machine)

```bash
scp root@<masternode1-ip>:/root/etcd-backup-*.db ./
scp -r root@<masternode1-ip>:/root/k8s-backup-* ./
```

---

## PHASE 1 — Upgrade 1.29.x → 1.29.15

### masternode1 (First Control Plane)

```bash
# Upgrade kubeadm
sudo apt-mark unhold kubeadm
sudo apt-get update && sudo apt-get install -y kubeadm=1.29.15-1.1
sudo apt-mark hold kubeadm
kubeadm version

# Plan and apply
sudo kubeadm upgrade plan v1.29.15
sudo kubeadm upgrade apply v1.29.15
# Type 'y' when prompted — wait for SUCCESS message

# Drain
kubectl drain masternode1 --ignore-daemonsets --delete-emptydir-data

# Upgrade kubelet and kubectl
sudo apt-mark unhold kubelet kubectl
sudo apt-get install -y kubelet=1.29.15-1.1 kubectl=1.29.15-1.1
sudo apt-mark hold kubelet kubectl
sudo systemctl daemon-reload && sudo systemctl restart kubelet

# Uncordon
kubectl uncordon masternode1
kubectl get nodes
```

### masternode2 (SSH into masternode2)

```bash
# On masternode2
sudo apt-mark unhold kubeadm
sudo apt-get update && sudo apt-get install -y kubeadm=1.29.15-1.1
sudo apt-mark hold kubeadm
sudo kubeadm upgrade node

# From masternode1 — drain
kubectl drain masternode2 --ignore-daemonsets --delete-emptydir-data

# On masternode2 — upgrade kubelet and kubectl
sudo apt-mark unhold kubelet kubectl
sudo apt-get install -y kubelet=1.29.15-1.1 kubectl=1.29.15-1.1
sudo apt-mark hold kubelet kubectl
sudo systemctl daemon-reload && sudo systemctl restart kubelet

# From masternode1 — uncordon
kubectl uncordon masternode2
kubectl get nodes
```

### masternode3 (SSH into masternode3)

```bash
# On masternode3
sudo apt-mark unhold kubeadm
sudo apt-get update && sudo apt-get install -y kubeadm=1.29.15-1.1
sudo apt-mark hold kubeadm
sudo kubeadm upgrade node

# From masternode1 — drain
kubectl drain masternode3 --ignore-daemonsets --delete-emptydir-data

# On masternode3 — upgrade kubelet and kubectl
sudo apt-mark unhold kubelet kubectl
sudo apt-get install -y kubelet=1.29.15-1.1 kubectl=1.29.15-1.1
sudo apt-mark hold kubelet kubectl
sudo systemctl daemon-reload && sudo systemctl restart kubelet

# From masternode1 — uncordon
kubectl uncordon masternode3
kubectl get nodes
```

### workernode1 (SSH into workernode1)

```bash
# On workernode1
sudo apt-mark unhold kubeadm
sudo apt-get update && sudo apt-get install -y kubeadm=1.29.15-1.1
sudo apt-mark hold kubeadm
sudo kubeadm upgrade node

# From masternode1 — drain
kubectl drain workernode1 --ignore-daemonsets --delete-emptydir-data

# On workernode1 — upgrade kubelet and kubectl
sudo apt-mark unhold kubelet kubectl
sudo apt-get install -y kubelet=1.29.15-1.1 kubectl=1.29.15-1.1
sudo apt-mark hold kubelet kubectl
sudo systemctl daemon-reload && sudo systemctl restart kubelet

# From masternode1 — uncordon
kubectl uncordon workernode1
kubectl get nodes
```

### workernode2 (SSH into workernode2)

```bash
# On workernode2
sudo apt-mark unhold kubeadm
sudo apt-get update && sudo apt-get install -y kubeadm=1.29.15-1.1
sudo apt-mark hold kubeadm
sudo kubeadm upgrade node

# From masternode1 — drain
kubectl drain workernode2 --ignore-daemonsets --delete-emptydir-data

# On workernode2 — upgrade kubelet and kubectl
sudo apt-mark unhold kubelet kubectl
sudo apt-get install -y kubelet=1.29.15-1.1 kubectl=1.29.15-1.1
sudo apt-mark hold kubelet kubectl
sudo systemctl daemon-reload && sudo systemctl restart kubelet

# From masternode1 — uncordon
kubectl uncordon workernode2
kubectl get nodes
```

### Verify all nodes on 1.29.15

```bash
kubectl get nodes
# All nodes should show VERSION = v1.29.15
```

---

## PHASE 2 — Upgrade 1.29.15 → 1.30.x

### Step 1 — Update apt repo to v1.30 on ALL nodes

Run the following on **every node** (masternode1/2/3, workernode1/2):

```bash
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key \
  | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] \
  https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /" \
  | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update

# Confirm 1.30 packages are visible
apt-cache madison kubeadm | grep 1.30
```

### masternode1 (First Control Plane)

```bash
# Upgrade kubeadm
sudo apt-mark unhold kubeadm
sudo apt-get install -y kubeadm=1.30.12-1.1
sudo apt-mark hold kubeadm
kubeadm version

# Plan and apply
sudo kubeadm upgrade plan
sudo kubeadm upgrade apply v1.30.12
# Type 'y' when prompted — wait for SUCCESS message

# Drain
kubectl drain masternode1 --ignore-daemonsets --delete-emptydir-data

# Upgrade kubelet and kubectl
sudo apt-mark unhold kubelet kubectl
sudo apt-get install -y kubelet=1.30.12-1.1 kubectl=1.30.12-1.1
sudo apt-mark hold kubelet kubectl
sudo systemctl daemon-reload && sudo systemctl restart kubelet

# Uncordon
kubectl uncordon masternode1
kubectl get nodes
```

### masternode2

```bash
# On masternode2
sudo apt-mark unhold kubeadm
sudo apt-get install -y kubeadm=1.30.12-1.1
sudo apt-mark hold kubeadm
sudo kubeadm upgrade node

# From masternode1
kubectl drain masternode2 --ignore-daemonsets --delete-emptydir-data

# On masternode2
sudo apt-mark unhold kubelet kubectl
sudo apt-get install -y kubelet=1.30.12-1.1 kubectl=1.30.12-1.1
sudo apt-mark hold kubelet kubectl
sudo systemctl daemon-reload && sudo systemctl restart kubelet

# From masternode1
kubectl uncordon masternode2
kubectl get nodes
```

### masternode3

```bash
# On masternode3
sudo apt-mark unhold kubeadm
sudo apt-get install -y kubeadm=1.30.12-1.1
sudo apt-mark hold kubeadm
sudo kubeadm upgrade node

# From masternode1
kubectl drain masternode3 --ignore-daemonsets --delete-emptydir-data

# On masternode3
sudo apt-mark unhold kubelet kubectl
sudo apt-get install -y kubelet=1.30.12-1.1 kubectl=1.30.12-1.1
sudo apt-mark hold kubelet kubectl
sudo systemctl daemon-reload && sudo systemctl restart kubelet

# From masternode1
kubectl uncordon masternode3
kubectl get nodes
```

### workernode1

```bash
# On workernode1
sudo apt-mark unhold kubeadm
sudo apt-get install -y kubeadm=1.30.12-1.1
sudo apt-mark hold kubeadm
sudo kubeadm upgrade node

# From masternode1
kubectl drain workernode1 --ignore-daemonsets --delete-emptydir-data

# On workernode1
sudo apt-mark unhold kubelet kubectl
sudo apt-get install -y kubelet=1.30.12-1.1 kubectl=1.30.12-1.1
sudo apt-mark hold kubelet kubectl
sudo systemctl daemon-reload && sudo systemctl restart kubelet

# From masternode1
kubectl uncordon workernode1
kubectl get nodes
```

### workernode2

```bash
# On workernode2
sudo apt-mark unhold kubeadm
sudo apt-get install -y kubeadm=1.30.12-1.1
sudo apt-mark hold kubeadm
sudo kubeadm upgrade node

# From masternode1
kubectl drain workernode2 --ignore-daemonsets --delete-emptydir-data

# On workernode2
sudo apt-mark unhold kubelet kubectl
sudo apt-get install -y kubelet=1.30.12-1.1 kubectl=1.30.12-1.1
sudo apt-mark hold kubelet kubectl
sudo systemctl daemon-reload && sudo systemctl restart kubelet

# From masternode1
kubectl uncordon workernode2
kubectl get nodes
```

---

## PHASE 3 — Final Verification

```bash
# All nodes should show v1.30.x and Ready status
kubectl get nodes

# Check all control plane pods are healthy
kubectl get pods -n kube-system

# Verify API server health
kubectl get --raw='/readyz?verbose'
```

### Expected output

```
NAME          STATUS   ROLES           AGE    VERSION
masternode1   Ready    control-plane   511d   v1.30.12
masternode2   Ready    control-plane   273d   v1.30.12
masternode3   Ready    control-plane   273d   v1.30.12
workernode1   Ready    <none>          511d   v1.30.12
workernode2   Ready    <none>          511d   v1.30.12
```

---

## What Gets Upgraded Automatically vs Manually

| Component | Upgraded By | Command |
|-----------|-------------|---------|
| kube-apiserver | kubeadm (auto) | `kubeadm upgrade apply` |
| kube-controller-manager | kubeadm (auto) | `kubeadm upgrade apply` |
| kube-scheduler | kubeadm (auto) | `kubeadm upgrade apply` |
| kube-proxy | kubeadm (auto) | `kubeadm upgrade apply` |
| CoreDNS | kubeadm (auto) | `kubeadm upgrade apply` |
| etcd | kubeadm (auto) | `kubeadm upgrade apply` |
| kubelet | **Manual** | `apt-get install kubelet` on each node |
| kubectl | **Manual** | `apt-get install kubectl` on each node |

---

## Backup Files Reference on the control plane (masternode1)

| File | Location | Contains |
|------|----------|----------|
| etcd snapshot | `/root/etcd-backup-<date>.db` | All cluster state |
| PKI certs | `/root/k8s-backup-<date>/pki/` | API server, etcd, SA certs |
| kubeconfig | `/root/k8s-backup-<date>/admin.conf` | Cluster admin access |
| etcd data dir | `/root/etcd-data-backup-<date>/` | Raw etcd data |


## Rollback Flow for your 3 control plane setup 
Run this first to get your exact etcd member details:
''''
kubectl exec -n kube-system etcd-masternode1 -- etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  member list
  '''

ALL 3 control planes simultaneously:
  
  1. Stop etcd + apiserver      ← move .yaml manifests out of /etc/kubernetes/manifests/
       ↓
  2. Copy same .db file          ← scp backup to masternode2 and masternode3
       ↓
  3. etcdctl snapshot restore    ← different --name & --peer-urls per node, SAME snapshot
       ↓
  4. Swap data dir               ← mv /var/lib/etcd-restore → /var/lib/etcd
       ↓
  5. Restore manifests           ← move .yaml files back
       ↓
  6. Verify kubectl get nodes
---

## References

- [Kubernetes Official Upgrade Docs](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/)
- [kubeadm Upgrade Reference](https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-upgrade/)
- [etcd Backup and Restore](https://etcd.io/docs/v3.5/op-guide/recovery/)
- [Kubernetes Version Skew Policy](https://kubernetes.io/releases/version-skew-policy/)
