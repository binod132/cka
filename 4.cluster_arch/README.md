## Prepare underlying infrastructure for installing a Kubernetes cluster

check hostname and /etc/hosts, test sudo ls working on all nodes

# change hostname if needed 
```sudo hostnamectl set-hostname correct-name```


# check containerd installed or not

# Installing kubeadm, kubelet and kubectl  (ALL nodes)
https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#check-network-adapters


## kubeadm init (log -v6)
I0216 05:43:04.891264   13558 initconfiguration.go:122] detected and using CRI socket: unix:///var/run/containerd/containerd.sock

I0216 05:43:05.013410   13558 checks.go:364] validating the contents of file /proc/sys/net/ipv4/ip_forward

I0216 05:43:04.904778   13558 checks.go:238] validating availability of port 6443
I0216 05:43:04.905697   13558 checks.go:238] validating availability of port 10259
I0216 05:43:04.905943   13558 checks.go:238] validating availability of port 10257

I0216 05:43:05.013492   13558 checks.go:238] validating availability of port 2379
I0216 05:43:05.013688   13558 checks.go:238] validating availability of port 2380

find error:         [ERROR KubeletVersion]: the kubelet version is higher than the control plane version. This is not a supported version skew and may lead to a malfunctional cluster. Kubelet version: "1.35.1" Control plane version: "1.34.1"


## Join command
kubeadm token create --print-join-command

### Optional
check kubelet     ```kubelet```
```apt list -a kubelet```
```kubelet --version```
```apt-cache madison kubelet```

### downgrade update
check kubernetes package list
```cat /etc/apt/sources.list.d/kubernetes.list```

# Kubernetes Downgrade Guide (v1.35.x → v1.34.1)

## Environment Assumptions

- OS: Ubuntu / Debian
- Package source: pkgs.k8s.io
- Current version installed: v1.35.x
- Target version: v1.34.1
- Fresh cluster (NOT yet initialized)

---

# 1️⃣ Verify Current Installed Versions

```bash
kubeadm version -o short
kubelet --version
kubectl version --client --short
```

Check installed packages:

```bash
apt list --installed | grep kube
```

If versions show `1.35.x`, continue.

---

# 2️⃣ Stop kubelet Before Downgrade

```bash
sudo systemctl stop kubelet
```

---

# 3️⃣ Check Current Kubernetes Repository

```bash
cat /etc/apt/sources.list.d/kubernetes.list
```

You will likely see:

```
deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.35/deb/ /
```

This means only 1.35 packages are available.

---

# 4️⃣ Switch Repository to v1.34

Edit the file:

```bash
sudo nano /etc/apt/sources.list.d/kubernetes.list
```

Change:

```
v1.35
```

To:

```
v1.34
```

Final content should be:

```
deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.34/deb/ /
```

Save and exit.

---

# 5️⃣ Update Package Index

```bash
sudo apt update
```

---

# 6️⃣ Verify 1.34 Versions Are Available

```bash
apt list -a kubelet
```

You should now see:

```
1.34.1-1.1
1.34.0-1.1
```

---

# 7️⃣ Downgrade All Components Together

⚠ Kubernetes components must match versions.

```bash
sudo apt install -y kubelet=1.34.1-1.1 kubeadm=1.34.1-1.1 kubectl=1.34.1-1.1 --allow-downgrades
```

---

# 8️⃣ Hold the Version (Prevent Auto-Upgrade)

```bash
sudo apt-mark hold kubelet kubeadm kubectl
```

---

# 9️⃣ Verify Downgrade

```bash
kubeadm version -o short
kubelet --version
kubectl version --client --short
```

All should display:

```
v1.34.1
```

---

# 🔟 Start and Enable kubelet

```bash
sudo systemctl start kubelet
sudo systemctl enable kubelet
```

---

# 1️⃣1️⃣ Choosing Pod Network CIDR

`--pod-network-cidr` defines the IP range used for Pods.

## Rules

- Must NOT overlap with:
  - Node IP network
  - Service CIDR
  - VPN network
  - Cloud VPC
- Must use private IP space:
  - 10.0.0.0/8
  - 172.16.0.0/12
  - 192.168.0.0/16

## Safe Lab Default (Recommended)

If your node IP is `192.168.x.x`, use:

```
10.244.0.0/16
```

This avoids conflicts and works well with Flannel.

---

# 1️⃣2️⃣ Initialize Kubernetes Control Plane

```bash
sudo kubeadm init \
--kubernetes-version=1.34.1 \
--pod-network-cidr=10.244.0.0/16 \
--ignore-preflight-errors=NumCPU \
--ignore-preflight-errors=Mem
```

### Flag Explanation

- `--kubernetes-version=1.34.1` → Forces specific version
- `--pod-network-cidr=10.244.0.0/16` → Pod IP range
- `--ignore-preflight-errors=NumCPU` → Ignore low CPU (lab only)
- `--ignore-preflight-errors=Mem` → Ignore low memory (lab only)

⚠ Do NOT use ignore flags in production.

---

# 1️⃣3️⃣ Configure kubectl for Current User

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Verify:

```bash
kubectl get nodes
```

Node will show `NotReady` until CNI is installed.

---

# 1️⃣4️⃣ Install CNI (Example: Flannel)

```bash
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
```

Wait 1–2 minutes, then:

```bash
kubectl get nodes
```

Node should become:

```
Ready
```

---

# ✅ Final Checklist

- Repository switched to v1.34
- kubelet/kubeadm/kubectl downgraded to v1.34.1
- Versions locked with apt-mark
- kubeadm init completed
- CNI installed
- Node status Ready

---

**End of Document**
