1. Set up the Infrastructure
First, you need three Ubuntu 22.04 machines (these could be VMs, cloud instances, or bare metal) with the following prerequisites:

2 CPUs or more
2GB of RAM or more
Full network connectivity between all machines in the cluster (public or private network is fine)
Unique hostname, MAC address, and product_uuid for every node. Use sudo hostnamectl set-hostname <node-name> to set hostnames like master-node, worker-node1, and worker-node2.
NOTE: These required ports need to be open. Check the official kubernetes documentation https://kubernetes.io/docs/reference/networking/ports-and-protocols/

Step 1: Preparing the Nodes
First, you'll need to prepare each of your three nodes. This includes disabling swap, enabling IP forwarding, and loading necessary kernel modules.
```bash
# Disable swap
swapoff -a
sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

# Load the overlay and br_netfilter modules
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# Set sysctl params, these will persist across reboots
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# Apply sysctl params without reboot
sudo sysctl --system
```

Step 2: Installing containerd
Install containerd as per Docker's official guide:
```bash
# Install prerequisites
sudo apt-get update
sudo apt-get install ca-certificates curl gnupg

# Add Docker’s official GPG key
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Add Docker’s repository
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install containerd
sudo apt-get update
sudo apt-get install containerd.io

# Start and enable containerd service
sudo systemctl start containerd
sudo systemctl enable containerd

# Configure containerd to use systemd as the cgroup driver
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml

# Modify the configuration file to set the systemd cgroup
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

# Restart containerd to apply the changes
sudo systemctl restart containerd
```

Step 3: Installing kubeadm, kubelet, and kubectl
You will follow the official Kubernetes guide to install kubeadm, kubelet, and kubectl:
```bash
# Update the apt package index and install packages needed to use the Kubernetes apt repository
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl

# Download the public signing key for the Kubernetes package repositories
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

# Add the Kubernetes apt repository
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

# Update apt package index, install kubelet, kubeadm, and kubectl, and pin their version
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```