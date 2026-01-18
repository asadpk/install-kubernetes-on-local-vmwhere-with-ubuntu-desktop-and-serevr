# Fixing Calico (BIRD / IP-in-IP) Issues Behind Firewall (UFW)

**Author:** asad@k8s-admin
**Environment:** Kubernetes (kubeadm), Calico CNI, Ubuntu-based VMs

---

## 📌 Problem Summary

In a multi-node Kubernetes cluster using **Calico** as the CNI, pods may stay in a **Running** state but Calico nodes remain **0/1 Ready**.

Typical symptoms:

* `calico-node` pods stuck at `0/1 Ready`
* Pod-to-pod communication fails across nodes
* Errors related to **BIRD** (Calico’s routing daemon)

Example error:

```
BIRD: unable to connect to BIRDv4 socket
Readiness probe failed: Number of node(s) with BGP peering established = 0
```

---

## 🔍 Root Cause

This issue usually happens when:

* **UFW (firewall)** is enabled on the host
* **IP-in-IP traffic (protocol 4)** used by Calico is blocked
* UFW does **not support IP-in-IP rules directly**

When IP-in-IP is blocked:

* `tunl0` interface is not created
* BIRD cannot establish routing
* Calico readiness probes fail

Disabling the firewall works temporarily because all traffic is allowed.

---

## ✅ Verified Working Configuration

* Calico running in **IP-in-IP mode**
* Correct node IP auto-detection
* Firewall enabled, but IP-in-IP allowed via iptables

Example environment inside Calico pod:

```
IP_AUTODETECTION_METHOD=interface=^ens33$
CALICO_IPV4POOL_IPIP=Always
```

---

## 🛠 Step-by-Step Fix (Permanent Solution)

> Run **all steps on BOTH master and worker nodes**

---

### 1️⃣ Identify the Correct Network Interface

Check active interfaces:

```
ip a
```

Example:

```
ens33: inet 192.168.230.128/24
```

Use this interface name in all commands below.

---

### 2️⃣ Ensure IPIP Kernel Module Is Loaded

```
sudo modprobe ipip
sudo sysctl -w net.ipv4.conf.all.forwarding=1
```

Verify:

```
lsmod | grep ipip
```

---

### 3️⃣ Configure Calico IP Auto-Detection

Run **only from the control-plane node**:

```
kubectl -n kube-system set env daemonset/calico-node \
  IP_AUTODETECTION_METHOD=interface=^ens33$
```

Restart Calico pods:

```
kubectl delete pod -n kube-system -l k8s-app=calico-node
```

---

### 4️⃣ Allow IP-in-IP Traffic Using iptables

⚠️ UFW does NOT support protocol 4 (IPIP). Use iptables instead.

```
sudo iptables -I INPUT  -i ens33 -p 4 -j ACCEPT
sudo iptables -I OUTPUT -o ens33 -p 4 -j ACCEPT
```

Persist rules:

```
sudo apt install -y iptables-persistent
sudo netfilter-persistent save
```

---

### 5️⃣ (Optional) Allow BGP Port (If Using BGP)

```
sudo ufw allow in on ens33 proto tcp to any port 179
```

---

### 6️⃣ Re-enable Firewall

```
sudo ufw enable
```

---

## 🔎 Verification

Check Calico status:

```
kubectl get pods -n kube-system
```

Expected output:

```
calico-node-xxxxx   1/1   Running
```

Check tunnel interface:

```
ip a | grep tunl0
```

Test pod networking:

```
kubectl exec -it test -- ping <pod-ip>
```
kubectl run test --image=busybox --restart=Never -- sleep 3600
kubectl exec -it test -- ping 8.8.8.8
---

## 📘 Why This Works

* Calico uses **IP-in-IP (protocol 4)** for pod traffic across nodes
* UFW cannot manage protocol 4 rules
* iptables allows low-level protocol handling
* BIRD starts successfully once tunneling works

---

## 🧠 Key Takeaways

* Never rely only on UFW for Calico IPIP traffic
* Always verify `tunl0` interface
* Firewall + Calico can coexist with proper iptables rules
* This setup is ideal for VM-based Kubernetes labs

---

## 🏷 Tags

`kubernetes` `calico` `cni` `ipip` `bird` `ufw` `iptables` `networking` `kubeadm` `vmware`

---

✅ **Status:** Tested & Working
🧑‍💻 **Maintainer:** asad@k8s-admin
