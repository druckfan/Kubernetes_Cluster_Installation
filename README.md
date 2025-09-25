# 🚀 Kubernetes Cluster Setup on Ubuntu 24.04 (Control Plane)

This guide describes how to set up a **Kubernetes control-plane node (master)** on Ubuntu 24.04 using **containerd** as the runtime and **Calico** as the CNI plugin.

---

## 📑 Table of Contents
1. [Prerequisites](#-prerequisites)  
2. [Set Hostname and Hosts File](#-1-set-hostname-and-hosts-file)  
3. [Disable Swap](#-2-disable-swap)  
4. [Enable Kernel Modules](#-3-enable-kernel-modules)  
5. [Configure Kernel Networking](#-4-configure-kernel-networking)  
6. [Install Container Runtime (containerd)](#-5-install-container-runtime-containerd)  
7. [Install Kubernetes Components](#-6-install-kubernetes-components)  
8. [Initialize the Control Plane](#-7-initialize-the-control-plane)  
9. [Configure kubectl Access](#-8-configure-kubectl-access)  
10. [Install Calico CNI (Pod Networking)](#-9-install-calico-cni-pod-networking)  
11. [Verify Cluster](#-10-verify-cluster)  
12. [Next Steps](#-next-steps)  

---

## 🔧 Prerequisites
- Ubuntu 24.04 installed on all nodes  
- A user with `sudo` privileges  
- Proper hostname resolution between master and worker nodes  

---

## 1️⃣ Set Hostname and Hosts File
```bash
sudo hostnamectl set-hostname "k8s-master-node"
sudo nano /etc/hosts
```

Example `/etc/hosts`:
```
192.168.1.100   k8s-master-node
192.168.1.101   k8s-worker-node-1
192.168.1.102   k8s-worker-node-2
```

Test connectivity:
```bash
ping -c 3 k8s-worker-node-1
```

---

## 2️⃣ Disable Swap
```bash
sudo swapoff -a
```
> 💡 To disable swap permanently, remove or comment out the swap entry in `/etc/fstab`.

---

## 3️⃣ Enable Kernel Modules
```bash
sudo modprobe overlay
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
```

---

## 4️⃣ Configure Kernel Networking
```bash
sudo tee /etc/sysctl.d/k8s.conf <<EOF
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
```

---

## 5️⃣ Install Container Runtime (containerd)
Install Docker dependencies:
```bash
sudo apt update
sudo apt install -y docker.io
sudo systemctl enable docker
```

Install and configure **containerd**:
```bash
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml > /dev/null

sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

sudo systemctl restart containerd
sudo systemctl enable containerd
sudo systemctl status containerd
```

---

## 6️⃣ Install Kubernetes Components
Add the Kubernetes APT repo:
```bash
sudo apt-get install -y curl ca-certificates apt-transport-https
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key |     sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /" |     sudo tee /etc/apt/sources.list.d/kubernetes.list
```

Install Kubernetes:
```bash
sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

---

## 7️⃣ Initialize the Control Plane
```bash
sudo kubeadm init --pod-network-cidr=10.20.0.0/16
```

> ⚠️ Save the `kubeadm join` command displayed at the end — you will need it for worker nodes.

---

## 8️⃣ Configure kubectl Access
For the current user:
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

---

## 9️⃣ Install Calico CNI (Pod Networking)
Deploy the Calico operator:
```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/tigera-operator.yaml
```

Download and customize resources:
```bash
curl -O https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/custom-resources.yaml
sed -i 's|cidr: 192.168.0.0/16|cidr: 10.20.0.0/16|' custom-resources.yaml
kubectl apply -f custom-resources.yaml
```

---

## 🔟 Verify Cluster
```bash
kubectl get nodes
kubectl get pods -A
```

✅ The master node should be **Ready** and Calico pods should be running.

---

## ▶️ Next Steps
- Set up **worker nodes** and join them using the `kubeadm join` command.  
- Deploy workloads and services on your new cluster.  

---
