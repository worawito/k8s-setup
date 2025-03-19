Kubernetes 1.32 on Rocky Linux 9.5

📌 Overview

This guide provides step-by-step instructions for installing a Kubernetes v1.32 cluster on Rocky Linux 9.5 using:

containerd as the container runtime

Cilium as the CNI (Container Networking Interface)

Kubeadm for cluster initialization

Three nodes: 1 master, 2 workers

🖥️ System Requirements

Component

Specification

OS

Rocky Linux 9.5

CPU

2+ vCPUs (each node)

RAM

4GB+ (each node)

Storage

20GB+ (each node)

Network

Static IPs in 192.168.1.0/24

⚙️ Step 1: Prepare All Nodes

Perform these steps on all nodes (Master & Workers).
```bash
#!/bin/bash
# Step 1: Update System
sudo dnf update -y

# Step 2: Set Hostname (Pass hostname as argument)
if [ -n "$1" ]; then
    sudo hostnamectl set-hostname $1
fi

# Step 3: Disable Swap
sudo swapoff -a
sudo sed -i '/swap/d' /etc/fstab

# Step 4: Enable Kernel Modules
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
sudo modprobe overlay && sudo modprobe br_netfilter

# Step 5: Configure Sysctl for Kubernetes Networking
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF
sudo sysctl --system

# Step 6: Install Containerd
sudo dnf install -y containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl enable --now containerd

# Step 7: Install Kubernetes Components
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.32/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.32/rpm/repodata/repomd.xml.key
EOF
sudo dnf install -y kubelet kubeadm kubectl
sudo systemctl enable --now kubelet

echo "✅ Common setup completed. Run the master or worker script next."

Run on all nodes:

chmod +x setup-common.sh
sudo ./setup-common.sh <node-hostname>
```bash

🚀 Step 2: Initialize the Master Node

Run this only on the master node.

#!/bin/bash
# Step 1: Open Firewall Ports for Master
sudo firewall-cmd --permanent --add-port=6443/tcp  # API Server
sudo firewall-cmd --permanent --add-port=2379-2380/tcp  # etcd
sudo firewall-cmd --permanent --add-port=10250/tcp  # Kubelet API
sudo firewall-cmd --permanent --add-port=10257/tcp  # Controller Manager
sudo firewall-cmd --permanent --add-port=10259/tcp  # Scheduler
sudo firewall-cmd --permanent --add-port=4240/tcp   # Cilium
sudo firewall-cmd --reload

# Step 2: Initialize Kubernetes Cluster
sudo kubeadm init --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=192.168.1.100

# Step 3: Setup Kubeconfig
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Step 4: Install Cilium CNI
kubectl create -f https://raw.githubusercontent.com/cilium/cilium/v1.14.1/install/kubernetes/quick-install.yaml

echo "✅ Master setup completed. Run 'kubeadm token create --print-join-command' to get the worker join command."

Run on the master node:

chmod +x setup-master.sh
sudo ./setup-master.sh

🤖 Step 3: Join Worker Nodes

Run this only on worker nodes.

#!/bin/bash
# Step 1: Open Firewall Ports for Worker
sudo firewall-cmd --permanent --add-port=10250/tcp  # Kubelet API
sudo firewall-cmd --permanent --add-port=30000-32767/tcp  # NodePort Services
sudo firewall-cmd --permanent --add-port=4240/tcp   # Cilium
sudo firewall-cmd --reload

echo "⚠️ Run 'kubeadm token create --print-join-command' on the master to get the join command."

Run on each worker node:

chmod +x setup-worker.sh
sudo ./setup-worker.sh

Then, join the worker to the cluster using the command from the master:

sudo kubeadm join <master-ip>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>

🔍 Step 4: Verify Cluster Status

Run the following commands on the master node:

kubectl get nodes -o wide
kubectl get pods -n kube-system

✅ Expected Output:

NAME     STATUS   ROLES           AGE     VERSION   INTERNAL-IP
master   Ready    control-plane   10m     v1.32.0   192.168.1.100
worker1  Ready    <none>          5m      v1.32.0   192.168.1.101
worker2  Ready    <none>          5m      v1.32.0   192.168.1.102

🏗️ Step 5: Deploy a Test Application

kubectl create deployment nginx --image=nginx --replicas=2
kubectl expose deployment nginx --port=80 --type=NodePort
kubectl get svc nginx

Test with:

curl http://<worker-ip>:<nodeport>

✅ You should see the Nginx welcome page!

🎯 Conclusion

Your Kubernetes v1.32 cluster on Rocky Linux 9.5 is now fully operational with containerd and Cilium! 🚀

If you have any issues, check logs:

kubectl logs -n kube-system <pod-name>
kubectl describe node <node-name>

Happy Kubernetes hacking! 🔥

