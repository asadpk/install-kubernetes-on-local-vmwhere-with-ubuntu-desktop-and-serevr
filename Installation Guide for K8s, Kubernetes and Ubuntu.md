# 🚀 Kubernetes Cluster Setup Guide

**On-Premises | Ubuntu | kubeadm | containerd**

A step-by-step, production-aligned guide to set up a **2-node Kubernetes cluster** (1 Control Plane + 1 Worker) using **kubeadm** and **containerd**. This guide is **GitHub-ready**, cleanly structured, and beginner-to-intermediate friendly.

---

## 📌 Table of Contents

1. [Architecture Overview](#-architecture-overview)
2. [Prerequisites](#-prerequisites)
3. [Initial System Setup](#-initial-system-setup)
4. [Container Runtime Installation (containerd)](#-container-runtime-installation-containerd)
5. [Kubernetes Components Installation](#-kubernetes-components-installation)
6. [Cluster Initialization](#-cluster-initialization)
7. [CNI Installation (Calico)](#-cni-installation-calico)
8. [Worker Node Join](#-worker-node-join)
9. [Cluster Verification](#-cluster-verification)
10. [Test Deployment](#-test-deployment)
11. [Final Outcome](#-final-outcome)

---

## 🏗 Architecture Overview

| Role                            | OS             | Purpose                                         |
| ------------------------------- | -------------- | ----------------------------------------------- |
| **Master Node (Control Plane)** | Ubuntu Desktop | API Server, Scheduler, Controller Manager, etcd |
| **Worker Node (Data Plane)**    | Ubuntu Server  | Runs application workloads (Pods)               |

📌 **Cluster Type:** On-Prem / Home Lab / Virtualized Environment

---

## ⚙ Prerequisites

### 🔹 Virtual Machines

Create **two VMs** (VMware / VirtualBox):

| Node   | Operating System |
| ------ | ---------------- |
| Master | Ubuntu Desktop   |
| Worker | Ubuntu Server    |

---

### 🔹 System Requirements (Both Nodes)

* Minimum **2 CPU cores**
* Minimum **2 GB RAM**
* Internet connectivity
* Unique hostname
* Same network (nodes must ping each other)

---

## 🛠 Initial System Setup

### STEP 1️⃣: Verify System Resources

**Run on both nodes**

```bash
lscpu | grep "^CPU(s):"
free -h
lsb_release -a
```

---

### STEP 2️⃣: Set Hostnames

**Master Node**

```bash
sudo hostnamectl set-hostname k8s-master
exec bash
```

**Worker Node**

```bash
sudo hostnamectl set-hostname k8s-worker
exec bash
```

Verify:

```bash
hostname
```

---

### STEP 3️⃣: Configure `/etc/hosts`

Get IPs:

```bash
ip a
```

Edit hosts file **on both nodes**:

```bash
sudo nano /etc/hosts
```

Add:

```
<MASTER-IP>   k8s-master
<WORKER-IP>   k8s-worker
```

---

### STEP 4️⃣: Enable Firewall & Open Required Ports

#### 🔹 Master Node

```bash
sudo ufw enable
sudo ufw allow ssh
sudo ufw allow 6443/tcp
sudo ufw allow 2379:2380/tcp
sudo ufw allow 10250/tcp
sudo ufw allow 10257/tcp
sudo ufw allow 10259/tcp
sudo ufw status
```

#### 🔹 Worker Node

```bash
sudo ufw enable
sudo ufw allow ssh
sudo ufw allow 10250/tcp
sudo ufw allow 30000:32767/tcp
sudo ufw status
```

---

### STEP 5️⃣: Enable SSH Access

**Install SSH (Both Nodes)**

```bash
sudo apt update
sudo apt install -y openssh-server
sudo systemctl enable ssh
sudo systemctl start ssh
```

**Passwordless SSH (Master → Worker)**

```bash
ssh-keygen
ssh-copy-id k8s-worker
```

Test:

```bash
ssh k8s-worker
```

---

### STEP 6️⃣: Disable Swap (MANDATORY)

**Run on both nodes**

```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab
free -h
```

---

### STEP 7️⃣: Enable Kernel Modules & Networking

**Run on both nodes**

```bash
sudo modprobe overlay
sudo modprobe br_netfilter
```

Create sysctl config:

```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
```

Apply:

```bash
sudo sysctl --system
sysctl net.ipv4.ip_forward
```

---

## 📦 Container Runtime Installation (containerd)

**Run on both nodes**

```bash
sudo apt update
sudo apt install -y ca-certificates curl gnupg lsb-release
```

Add Docker repo:

```bash
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list
```

Install containerd:

```bash
sudo apt update
sudo apt install -y containerd.io
```

Configure containerd:

```bash
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
```

Edit config:

```bash
sudo nano /etc/containerd/config.toml
```

Change:

```toml
SystemdCgroup = true
```

Restart:

```bash
sudo systemctl restart containerd
sudo systemctl enable containerd
```

---

## ☸ Kubernetes Components Installation

**Run on both nodes**

```bash
sudo apt update
sudo apt install -y apt-transport-https ca-certificates curl
```

Add Kubernetes repo:

```bash
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

Install tools:

```bash
sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
sudo systemctl enable kubelet
```

---

## 🚦 Cluster Initialization

### 🔹 Master Node ONLY

```bash
sudo kubeadm init --pod-network-cidr=192.168.0.0/16
```

📌 **Save the join command** displayed at the end.

Configure kubectl:

```bash
mkdir -p $HOME/.kube
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Verify:

```bash
kubectl get nodes
```

---

## 🌐 CNI Installation (Calico)

**Master Node ONLY**

```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/calico.yaml
```

Wait:

```bash
kubectl get pods -n kube-system
```

---

## ➕ Worker Node Join

**Worker Node ONLY**

```bash
sudo kubeadm join <MASTER-IP>:6443 --token <token> \
--discovery-token-ca-cert-hash sha256:<hash>
```

---

## ✅ Cluster Verification

**Master Node**

```bash
kubectl get nodes
```

Expected:

```text
k8s-master   Ready
k8s-worker   Ready
```

---

## 🧪 Test Deployment

```bash
kubectl create deployment nginx --image=nginx
kubectl expose deployment nginx --type=NodePort --port=80
kubectl get svc
```

Access:

```
http://<WORKER-IP>:<NODEPORT>
```

---

## 🎉 Final Outcome

✅ Fully functional Kubernetes cluster
✅ Control Plane & Worker connected
✅ Containerd runtime configured
✅ Networking enabled via Calico
✅ Ready for real-world workloads

---

### ⭐ If this guide helped you

* Star ⭐ the repository
* Fork 🍴 for your lab
* Share with fellow DevOps learners

Happy Kubernetes-ing! ☸️
