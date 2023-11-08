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
```S
