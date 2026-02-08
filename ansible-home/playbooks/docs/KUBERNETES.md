# Kubernetes Cluster Setup

This Ansible configuration sets up a Kubernetes cluster for homelab learning purposes.

## Cluster Configuration

- **Control Plane**: kc01 (10.0.0.100)
- **Worker Nodes**: kn01 (10.0.0.101), kn02 (10.0.0.102)
- **Kubernetes Version**: 1.29
- **Container Runtime**: containerd
- **CNI Plugin**: Flannel (Pod CIDR: 10.244.0.0/16)

## Prerequisites

- Debian-based Linux on all nodes (Debian 12 or Ubuntu 22.04+)
- SSH access configured with ansible user
- Sudo privileges for ansible user
- At least 2 GB RAM per node
- At least 2 CPUs per node

## Installation

Run the complete cluster installation with:

```bash
cd ansible-home
ansible-playbook -i inventory/inventory.ini playbooks/install-k8s-cluster.yml
```

This will:
1. Install prerequisites on all nodes (containerd, kubeadm, kubelet, kubectl)
2. Initialize the control plane on kc01
3. Install Flannel CNI
4. Join worker nodes kn01 and kn02 to the cluster
5. Verify cluster status

## Individual Playbooks

You can also run the playbooks separately:

### 1. Install prerequisites only
```bash
ansible-playbook -i inventory/inventory.ini playbooks/install-k8s-prerequisites.yml
```

### 2. Setup cluster (after prerequisites)
```bash
ansible-playbook -i inventory/inventory.ini playbooks/setup-k8s-cluster.yml
```

### 3. Retrieve kubeconfig
```bash
ansible-playbook -i inventory/inventory.ini playbooks/get-kubeconfig.yml
```

This will save the kubeconfig to `./kubeconfig` in the ansible-home directory.

## Accessing the Cluster

### Option 1: From your machine
```bash
ansible-playbook -i inventory/inventory.ini playbooks/get-kubeconfig.yml
export KUBECONFIG=$(pwd)/kubeconfig
kubectl get nodes
```

### Option 2: SSH to control plane
```bash
ssh ansible@10.0.0.100
kubectl get nodes
```

## Common Operations

### Check cluster status
```bash
kubectl get nodes
kubectl get pods -A
kubectl cluster-info
```

### Deploy a test application
```bash
kubectl create deployment nginx --image=nginx
kubectl expose deployment nginx --port=80 --type=NodePort
kubectl get services
```

### Reset the cluster (if needed)
On each node:
```bash
sudo kubeadm reset -f
sudo rm -rf /etc/cni/net.d
sudo rm -rf ~/.kube
```

Then re-run the installation playbook.

## Troubleshooting

### Nodes not ready
Check pod networking:
```bash
kubectl get pods -n kube-flannel
kubectl logs -n kube-flannel <pod-name>
```

### Join command expired
Generate new join command on control plane:
```bash
kubeadm token create --print-join-command
```

### Check kubelet logs
On any node:
```bash
sudo journalctl -u kubelet -f
```

## Learning Resources

- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)
- [Kubernetes Tutorials](https://kubernetes.io/docs/tutorials/)

## Next Steps

1. Install kubectl on your local machine
2. Practice deploying applications
3. Learn about Kubernetes objects (Pods, Deployments, Services)
4. Explore Helm for package management
5. Try setting up monitoring with Prometheus/Grafana
