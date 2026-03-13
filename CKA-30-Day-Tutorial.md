# CKA Exam Preparation: 30-Day Comprehensive Tutorial

## Table of Contents

- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Environment Setup](#environment-setup)
- [30-Day Learning Path](#30-day-learning-path)
- [Week 1: Cluster Architecture & Installation](#week-1-cluster-architecture--installation)
- [Week 2: Workloads & Scheduling](#week-2-workloads--scheduling)
- [Week 3: Services, Networking & Storage](#week-3-services-networking--storage)
- [Week 4: Security, Troubleshooting & Review](#week-4-security-troubleshooting--review)
- [Exam Tips](#exam-tips)
- [Additional Resources](#additional-resources)

---

## Overview

This tutorial is designed for DevOps engineers preparing for the **Certified Kubernetes Administrator (CKA)** exam with 1 hour of daily study over 30 days.

**Environment:**

- Ubuntu 24.04 LTS
- Kubernetes 1.31
- containerd runtime
- Calico CNI
- 1 control plane node (2 vCPU, 4GB RAM)
- 2 worker nodes (4 vCPU, 8GB RAM each)

### CKA Exam Domains

1. **Cluster Architecture, Installation & Configuration** (25%) - Week 1
2. **Workloads & Scheduling** (15%) - Week 2
3. **Services & Networking** (20%) - Week 3
4. **Storage** (10%) - Week 3
5. **Troubleshooting** (30%) - Week 4

---

## Prerequisites

### Required Knowledge

- Linux command line proficiency
- Basic networking (IP, DNS, routing)
- Container fundamentals
- YAML syntax
- SSH and system administration

### Tools Needed

- Ansible 2.15+
- SSH access to VMs
- kubectl (installed via automation)

---

## Environment Setup with Ansible

### Directory Structure

```shell
cka-lab/
├── ansible/
│   ├── inventory.ini
│   ├── ansible.cfg
│   ├── site.yml
│   ├── group_vars/
│   │   └── all.yml
│   └── roles/
│       ├── common/
│       ├── containerd/
│       ├── kubernetes/
│       ├── control_plane/
│       └── worker/
└── exercises/
```

### Step 1: Create Inventory

```ini
# ansible/inventory.ini
[control_plane]
k8s-control ansible_host=192.168.1.10 ansible_user=ubuntu

[workers]
k8s-worker1 ansible_host=192.168.1.11 ansible_user=ubuntu
k8s-worker2 ansible_host=192.168.1.12 ansible_user=ubuntu

[all:vars]
ansible_python_interpreter=/usr/bin/python3

[k8s_cluster:children]
control_plane
workers
```

### Step 2: Configuration Files

**ansible.cfg:**

```ini
[defaults]
inventory = inventory.ini
host_key_checking = False
roles_path = ./roles

[privilege_escalation]
become = True
become_method = sudo
```

**group_vars/all.yml:**

```yaml
---
kubernetes_version: "1.31"
kubernetes_version_patch: "1.31.0-1.1"
pod_network_cidr: "10.244.0.0/16"
service_cidr: "10.96.0.0/12"
calico_version: "v3.28.0"
disable_swap: true
```

### Step 3: Ansible Roles

**roles/common/tasks/main.yml:**

```yaml
---
- name: Update apt cache
  apt:
    update_cache: yes

- name: Install common packages
  apt:
    name:
      - apt-transport-https
      - ca-certificates
      - curl
      - gnupg
      - vim
      - bash-completion
    state: present

- name: Disable swap
  shell: swapoff -a && sed -i '/ swap / s/^/#/' /etc/fstab
  when: disable_swap | bool

- name: Load kernel modules
  modprobe:
    name: "{{ item }}"
  loop:
    - overlay
    - br_netfilter

- name: Configure sysctl
  sysctl:
    name: "{{ item.name }}"
    value: "{{ item.value }}"
    sysctl_file: /etc/sysctl.d/k8s.conf
    reload: yes
  loop:
    - { name: 'net.bridge.bridge-nf-call-iptables', value: '1' }
    - { name: 'net.ipv4.ip_forward', value: '1' }

- name: Update /etc/hosts
  template:
    src: hosts.j2
    dest: /etc/hosts
```

**roles/common/templates/hosts.j2:**

```jinja2
127.0.0.1 localhost

{% for host in groups['k8s_cluster'] %}
{{ hostvars[host]['ansible_host'] }} {{ host }}
{% endfor %}
```

**roles/containerd/tasks/main.yml:**

```yaml
---
- name: Install containerd
  apt:
    name: containerd
    state: present

- name: Create containerd config directory
  file:
    path: /etc/containerd
    state: directory

- name: Generate default config
  shell: containerd config default > /etc/containerd/config.toml
  args:
    creates: /etc/containerd/config.toml

- name: Enable systemd cgroup
  replace:
    path: /etc/containerd/config.toml
    regexp: 'SystemdCgroup = false'
    replace: 'SystemdCgroup = true'
  notify: restart containerd

- name: Enable containerd
  systemd:
    name: containerd
    enabled: yes
    state: started
```

**roles/containerd/handlers/main.yml:**

```yaml
---
- name: restart containerd
  systemd:
    name: containerd
    state: restarted
```

**roles/kubernetes/tasks/main.yml:**

```yaml
---
- name: Add Kubernetes GPG key
  apt_key:
    url: https://pkgs.k8s.io/core:/stable:/v{{ kubernetes_version }}/deb/Release.key
    keyring: /etc/apt/keyrings/kubernetes-apt-keyring.gpg

- name: Add Kubernetes repository
  apt_repository:
    repo: "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v{{ kubernetes_version }}/deb/ /"
    filename: kubernetes

- name: Install Kubernetes components
  apt:
    name:
      - kubelet={{ kubernetes_version_patch }}
      - kubeadm={{ kubernetes_version_patch }}
      - kubectl={{ kubernetes_version_patch }}
    state: present
    update_cache: yes

- name: Hold Kubernetes packages
  dpkg_selections:
    name: "{{ item }}"
    selection: hold
  loop:
    - kubelet
    - kubeadm
    - kubectl

- name: Enable kubelet
  systemd:
    name: kubelet
    enabled: yes
```

**roles/control_plane/tasks/main.yml:**

```yaml
---
- name: Check if cluster initialized
  stat:
    path: /etc/kubernetes/admin.conf
  register: kubeadm_init

- name: Initialize Kubernetes cluster
  command: >
    kubeadm init
    --pod-network-cidr={{ pod_network_cidr }}
    --service-cidr={{ service_cidr }}
    --kubernetes-version={{ kubernetes_version }}.0
  when: not kubeadm_init.stat.exists

- name: Create .kube directory
  file:
    path: /home/{{ ansible_user }}/.kube
    state: directory
    owner: "{{ ansible_user }}"

- name: Copy admin.conf
  copy:
    src: /etc/kubernetes/admin.conf
    dest: /home/{{ ansible_user }}/.kube/config
    owner: "{{ ansible_user }}"
    remote_src: yes

- name: Install Calico CNI
  become_user: "{{ ansible_user }}"
  shell: |
    kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/{{ calico_version }}/manifests/tigera-operator.yaml
    kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/{{ calico_version }}/manifests/custom-resources.yaml
  when: not kubeadm_init.stat.exists

- name: Generate join command
  command: kubeadm token create --print-join-command
  register: join_command_raw

- name: Save join command
  copy:
    content: "{{ join_command_raw.stdout }}"
    dest: /tmp/join-command.sh
    mode: '0755'

- name: Fetch join command
  fetch:
    src: /tmp/join-command.sh
    dest: ./join-command.sh
    flat: yes

- name: Enable bash completion
  lineinfile:
    path: /home/{{ ansible_user }}/.bashrc
    line: "{{ item }}"
  loop:
    - 'source <(kubectl completion bash)'
    - 'alias k=kubectl'
    - 'complete -o default -F __start_kubectl k'
```

**roles/worker/tasks/main.yml:**

```yaml
---
- name: Check if node joined
  stat:
    path: /etc/kubernetes/kubelet.conf
  register: kubelet_conf

- name: Copy join command
  copy:
    src: ./join-command.sh
    dest: /tmp/join-command.sh
    mode: '0755'
  when: not kubelet_conf.stat.exists

- name: Join cluster
  command: /tmp/join-command.sh
  when: not kubelet_conf.stat.exists
```

**site.yml:**

```yaml
---
- name: Setup all nodes
  hosts: k8s_cluster
  roles:
    - common
    - containerd
    - kubernetes

- name: Setup control plane
  hosts: control_plane
  roles:
    - control_plane

- name: Join workers
  hosts: workers
  roles:
    - worker

- name: Verify cluster
  hosts: control_plane
  become_user: ubuntu
  tasks:
    - name: Wait for nodes
      command: kubectl get nodes
      register: nodes
      until: nodes.stdout.find("NotReady") == -1
      retries: 20
      delay: 15

    - name: Show cluster status
      command: kubectl get nodes -o wide
      register: status

    - debug: var=status.stdout_lines
```

### Step 4: Deploy

```bash
cd cka-lab/ansible
ansible all -m ping
ansible-playbook site.yml
```

Installation takes ~10-15 minutes.

### Verification

```bash
ssh ubuntu@192.168.1.10
kubectl get nodes
kubectl get pods -A
kubectl run test --image=nginx
kubectl get pods
kubectl delete pod test
```

---

## Week 1: Cluster Architecture & Installation (25% of exam)

### Day 1: Kubernetes Architecture Deep Dive

**Core Components:**

**Control Plane:**

- **kube-apiserver**: REST API frontend, authentication/authorization
- **etcd**: Key-value store for all cluster data
- **kube-scheduler**: Assigns pods to nodes
- **kube-controller-manager**: Runs controllers (Node, Job, Endpoint, ServiceAccount)
- **cloud-controller-manager**: Cloud provider integration (optional)

**Node Components:**

- **kubelet**: Ensures containers are running
- **kube-proxy**: Network proxy, implements Services
- **Container Runtime**: containerd (we're using this), CRI-O, Docker

**Key Concepts:**

- Pod: Smallest unit (1+ containers)
- Node: Worker machine
- Cluster: Set of nodes
- Namespace: Virtual cluster isolation

**Hands-on Lab:**

```bash
# Create practice namespace
kubectl create namespace day1-practice

# Explore cluster components
kubectl get pods -n kube-system
kubectl get nodes -o wide
kubectl describe node k8s-worker1

# View API resources
kubectl api-resources
kubectl explain pod
kubectl explain pod.spec.containers

# Create a simple pod
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  namespace: day1-practice
spec:
  containers:
  - name: nginx
    image: nginx:1.25
    ports:
    - containerPort: 80
EOF

# Inspect the pod
kubectl get pod nginx-pod -n day1-practice -o wide
kubectl describe pod nginx-pod -n day1-practice
kubectl logs nginx-pod -n day1-practice
kubectl exec nginx-pod -n day1-practice -- nginx -v

# Cleanup
kubectl delete pod nginx-pod -n day1-practice
```

**Review Questions:**

1. What stores all cluster state?
2. Which component schedules pods?
3. What does kubelet do?
4. Name the control plane components.
5. What's the smallest deployable unit?

---

### Day 2: kubectl Mastery

**Essential Commands:**

```bash
# Viewing
kubectl get <resource> [-o wide|yaml|json]
kubectl describe <resource> <name>

# Creating
kubectl run <name> --image=<image>
kubectl create -f <file>
kubectl apply -f <file>

# Editing
kubectl edit <resource> <name>
kubectl patch <resource> <name>

# Deleting
kubectl delete <resource> <name>

# Debugging
kubectl logs <pod>
kubectl exec -it <pod> -- <command>
```

**Output Formats:**

- `-o wide`: Extended info
- `-o yaml`: Full specification
- `-o json`: JSON format
- `-o name`: Names only
- `-o jsonpath`: Custom queries

**Hands-on Lab:**

```bash
kubectl create namespace day2-practice

# Imperative pod creation
kubectl run nginx-imp --image=nginx:1.25 -n day2-practice

# Declarative (generate YAML)
kubectl run nginx-dec --image=nginx:1.25 --dry-run=client -o yaml > pod.yaml
kubectl apply -f pod.yaml -n day2-practice

# Labels and selectors
kubectl label pod nginx-imp env=test -n day2-practice
kubectl get pods -l env=test -n day2-practice

# JSONPath queries
kubectl get pod nginx-imp -n day2-practice -o jsonpath='{.status.podIP}'
echo ""
kubectl get pods -n day2-practice -o custom-columns=NAME:.metadata.name,IP:.status.podIP

# Edit resources
kubectl set image pod/nginx-imp nginx=nginx:1.26 -n day2-practice

# Generate YAML for other resources
kubectl create deployment web --image=nginx --dry-run=client -o yaml
kubectl create service clusterip mysvc --tcp=80:80 --dry-run=client -o yaml

# Explain (built-in docs)
kubectl explain pod.spec.containers

# Cleanup
kubectl delete namespace day2-practice
```

**Key Shortcuts:**

- `k` alias for kubectl
- `-n` for namespace
- `-A` for all namespaces
- `--dry-run=client -o yaml` for YAML generation

**Review Questions:**

1. Difference between `create` and `apply`?
2. How to generate YAML without creating resource?
3. How to filter by labels?
4. What's the shorthand for `--all-namespaces`?

---

### Day 3: etcd Backup and Restore

**Why Critical:**

- etcd stores ALL cluster state
- Losing etcd = losing cluster
- Required skill for CKA exam

**Backup Command:**

```bash
ETCDCTL_API=3 etcdctl snapshot save <file> \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

**Restore Process:**

1. Stop API server
2. Restore snapshot to new directory
3. Update etcd manifest
4. Start API server
5. Verify cluster

**Hands-on Lab:**

```bash
# SSH to control plane
ssh k8s-control

# Create test resources
kubectl create namespace backup-test
kubectl run pod1 --image=nginx -n backup-test
kubectl run pod2 --image=nginx -n backup-test

# Take backup
sudo mkdir -p /backup
export ETCDCTL_API=3

sudo etcdctl snapshot save /backup/etcd-backup-$(date +%Y%m%d).db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Verify backup
BACKUP_FILE=$(ls -t /backup/etcd-backup-*.db | head -1)
sudo etcdctl snapshot status $BACKUP_FILE --write-out=table

# Simulate disaster
kubectl delete namespace backup-test

# Restore process
sudo mv /etc/kubernetes/manifests/kube-apiserver.yaml /tmp/
sleep 10

sudo etcdctl snapshot restore $BACKUP_FILE \
  --data-dir=/var/lib/etcd-restore

sudo sed -i 's|/var/lib/etcd|/var/lib/etcd-restore|g' \
  /etc/kubernetes/manifests/etcd.yaml

sudo mv /tmp/kube-apiserver.yaml /etc/kubernetes/manifests/

# Wait for cluster
until kubectl get nodes &> /dev/null; do sleep 5; done

# Verify restoration
kubectl get namespace backup-test
kubectl get pods -n backup-test
```

**Backup Script:**

```bash
#!/bin/bash
# Save as /home/ubuntu/etcd-backup.sh

BACKUP_DIR="/backup"
DATE=$(date +%Y%m%d-%H%M%S)

export ETCDCTL_API=3

etcdctl snapshot save ${BACKUP_DIR}/etcd-${DATE}.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

if [ $? -eq 0 ]; then
  echo "Backup successful: etcd-${DATE}.db"
  # Keep last 7 backups
  cd ${BACKUP_DIR}
  ls -t etcd-*.db | tail -n +8 | xargs -r rm
else
  echo "Backup failed!"
  exit 1
fi
```

**Review Questions:**

1. Where is etcd data stored?
2. What certificates are needed?
3. Why stop API server during restore?
4. How to verify backup integrity?

---

### Day 4: Cluster Upgrades

**Upgrade Strategy:**

1. Backup etcd
2. Upgrade control plane
3. Upgrade worker nodes (one at a time)
4. Verify functionality

**Version Skew Policy:**

- Can't skip minor versions (1.29 → 1.30 → 1.31)
- kubelet can be 2 versions behind apiserver
- kubectl can be ±1 version

**Control Plane Upgrade:**

```bash
# Upgrade kubeadm
sudo apt-mark unhold kubeadm
sudo apt install -y kubeadm=1.31.0-1.1
sudo apt-mark hold kubeadm

# Plan upgrade
sudo kubeadm upgrade plan

# Apply upgrade
sudo kubeadm upgrade apply v1.31.0

# Drain node
kubectl drain k8s-control --ignore-daemonsets

# Upgrade kubelet/kubectl
sudo apt-mark unhold kubelet kubectl
sudo apt install -y kubelet=1.31.0-1.1 kubectl=1.31.0-1.1
sudo apt-mark hold kubelet kubectl

# Restart kubelet
sudo systemctl daemon-reload
sudo systemctl restart kubelet

# Uncordon
kubectl uncordon k8s-control
```

**Worker Node Upgrade:**

```bash
# On control plane: drain worker
kubectl drain k8s-worker1 --ignore-daemonsets --delete-emptydir-data

# On worker node
sudo apt-mark unhold kubeadm
sudo apt install -y kubeadm=1.31.0-1.1
sudo apt-mark hold kubeadm

sudo kubeadm upgrade node

sudo apt-mark unhold kubelet kubectl
sudo apt install -y kubelet=1.31.0-1.1 kubectl=1.31.0-1.1
sudo apt-mark hold kubelet kubectl

sudo systemctl daemon-reload
sudo systemctl restart kubelet

# On control plane: uncordon
kubectl uncordon k8s-worker1

# Repeat for worker2
```

**Verification:**

```bash
kubectl get nodes
kubectl version
kubectl get pods -A
```

**Review Questions:**

1. What's the correct upgrade order?
2. Can you skip versions?
3. What does drain do?
4. Why uncordon nodes?

---

### Day 5: Node Management

**Node Operations:**

- **Cordon**: Mark unschedulable (existing pods stay)
- **Uncordon**: Mark schedulable
- **Drain**: Evict pods + cordon
- **Delete**: Remove from cluster

**Labels and Taints:**

```bash
# Add label
kubectl label node worker1 disktype=ssd

# Add taint
kubectl taint node worker1 key=value:NoSchedule

# Remove taint
kubectl taint node worker1 key:NoSchedule-
```

**Hands-on Lab:**

```bash
# Label nodes
kubectl label node k8s-worker1 disktype=ssd environment=prod
kubectl label node k8s-worker2 disktype=hdd environment=dev

# View labels
kubectl get nodes --show-labels
kubectl get nodes -L disktype,environment

# Schedule pod with nodeSelector
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: nginx-ssd
spec:
  nodeSelector:
    disktype: ssd
  containers:
  - name: nginx
    image: nginx
EOF

kubectl get pod nginx-ssd -o wide

# Node affinity (advanced)
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: nginx-affinity
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: disktype
            operator: In
            values:
            - ssd
            - nvme
  containers:
  - name: nginx
    image: nginx
EOF

# Taints and tolerations
kubectl taint node k8s-worker1 maintenance=true:NoSchedule

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: nginx-toleration
spec:
  tolerations:
  - key: "maintenance"
    operator: "Equal"
    value: "true"
    effect: "NoSchedule"
  containers:
  - name: nginx
    image: nginx
EOF

# Cordon/drain practice
kubectl create deployment test --image=nginx --replicas=6
kubectl get pods -o wide

kubectl cordon k8s-worker1
kubectl get nodes

kubectl drain k8s-worker2 --ignore-daemonsets --delete-emptydir-data
kubectl get pods -o wide

kubectl uncordon k8s-worker1
kubectl uncordon k8s-worker2

# Cleanup
kubectl delete deployment test
kubectl delete pod nginx-ssd nginx-affinity nginx-toleration
kubectl taint node k8s-worker1 maintenance:NoSchedule-
```

**Review Questions:**

1. Difference between cordon and drain?
2. Three taint effects?
3. How to remove labels?
4. nodeSelector vs node affinity?

---

### Day 6: RBAC (Role-Based Access Control)

**RBAC Components:**

1. **Role**: Namespace permissions
2. **ClusterRole**: Cluster-wide permissions
3. **RoleBinding**: Grants Role to subjects
4. **ClusterRoleBinding**: Grants ClusterRole to subjects

**Structure:**

```yaml
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
```

**Common Verbs:**

- get, list, watch (read)
- create, update, patch (write)
- delete, deletecollection (delete)

**Hands-on Lab:**

```bash
kubectl create namespace rbac-test

# Create service account
kubectl create serviceaccount app-reader -n rbac-test

# Create role
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: rbac-test
rules:
- apiGroups: [""]
  resources: ["pods", "pods/log"]
  verbs: ["get", "list", "watch"]
EOF

# Create rolebinding
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: rbac-test
subjects:
- kind: ServiceAccount
  name: app-reader
  namespace: rbac-test
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
EOF

# Test permissions
kubectl auth can-i get pods -n rbac-test \
  --as system:serviceaccount:rbac-test:app-reader
# Returns: yes

kubectl auth can-i create pods -n rbac-test \
  --as system:serviceaccount:rbac-test:app-reader
# Returns: no

# Create pod using service account
kubectl run nginx --image=nginx -n rbac-test
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
  namespace: rbac-test
spec:
  serviceAccountName: app-reader
  containers:
  - name: kubectl
    image: bitnami/kubectl:latest
    command: ["sleep", "infinity"]
EOF

# Test from within pod
kubectl exec -it test-pod -n rbac-test -- kubectl get pods
kubectl exec -it test-pod -n rbac-test -- kubectl run test --image=nginx
# First succeeds, second fails

# ClusterRole for nodes
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-viewer
rules:
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get", "list"]
EOF

cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: view-nodes
subjects:
- kind: ServiceAccount
  name: app-reader
  namespace: rbac-test
roleRef:
  kind: ClusterRole
  name: node-viewer
  apiGroup: rbac.authorization.k8s.io
EOF

kubectl exec -it test-pod -n rbac-test -- kubectl get nodes
# Now succeeds

# Cleanup
kubectl delete namespace rbac-test
kubectl delete clusterrole node-viewer
kubectl delete clusterrolebinding view-nodes
```

**Review Questions:**

1. Role vs ClusterRole difference?
2. What are the four RBAC components?
3. How to test permissions?
4. Can RoleBinding reference ClusterRole?

---

### Day 7: Week 1 Review

**Practice Scenarios:**

**Scenario 1: Cluster Backup (10 min)**

```bash
# Check health
kubectl get nodes
kubectl get pods -A

# Create backup
sudo etcdctl snapshot save /backup/review-backup.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Verify
sudo etcdctl snapshot status /backup/review-backup.db --write-out=table
```

**Scenario 2: Node Maintenance (8 min)**

```bash
kubectl create deployment web --image=nginx --replicas=4
kubectl get pods -o wide

kubectl drain k8s-worker1 --ignore-daemonsets --delete-emptydir-data
kubectl get pods -o wide

kubectl uncordon k8s-worker1
kubectl delete deployment web
```

**Scenario 3: RBAC Setup (12 min)**

```bash
kubectl create namespace dev
kubectl create serviceaccount developer -n dev

cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: dev-role
  namespace: dev
rules:
- apiGroups: ["", "apps"]
  resources: ["pods", "deployments"]
  verbs: ["get", "list", "create", "delete"]
EOF

kubectl create rolebinding dev-binding \
  --role=dev-role \
  --serviceaccount=dev:developer \
  -n dev

kubectl auth can-i create pods -n dev \
  --as system:serviceaccount:dev:developer
```

**Speed Drills (practice until automatic):**

```bash
# 5 seconds each
kubectl run nginx --image=nginx
kubectl expose pod nginx --port=80
kubectl scale deployment app --replicas=5
kubectl run test --image=nginx --dry-run=client -o yaml
kubectl drain node --ignore-daemonsets --delete-emptydir-data
```

**Self-Assessment:**
Rate 1-5:

- Architecture: ___
- kubectl: ___
- etcd: ___
- Upgrades: ___
- Nodes: ___
- RBAC: ___

Target: 4+ on all topics

---

## Week 2: Workloads & Scheduling (15% of exam)

### Day 8: Pods and Pod Lifecycle

**Topics:**

- Pod phases (Pending, Running, Succeeded, Failed, Unknown)
- Container states (Waiting, Running, Terminated)
- Init containers
- Liveness/Readiness/Startup probes
- Restart policies
- Multi-container patterns

**Key Labs:**

- Multi-container pods (sidecar pattern)
- Init containers for setup
- Configuring probes
- Resource requests/limits

### Day 9: Deployments and ReplicaSets

**Topics:**

- Deployment strategies (RollingUpdate, Recreate)
- Scaling deployments
- Rolling updates and rollbacks
- ReplicaSet management
- Deployment status

**Key Labs:**

- Creating deployments
- Rolling updates
- Rollback procedures
- Scaling operations
- Update strategies

### Day 10: StatefulSets and DaemonSets

**Topics:**

- StatefulSet use cases
- Stable network identities
- Ordered deployment/scaling
- DaemonSet scheduling
- Node selection for DaemonSets

**Key Labs:**

- Deploy StatefulSet with persistent storage
- DaemonSet for logging agents
- Update strategies
- Headless services

### Day 11: Jobs and CronJobs

**Topics:**

- Job patterns (single, parallel, work queue)
- Job completion and parallelism
- CronJob schedules
- Job failure handling
- TTL for finished jobs

**Key Labs:**

- Create batch jobs
- Parallel job execution
- Schedule CronJobs
- Clean up completed jobs

### Day 12: Resource Management

**Topics:**

- Resource requests and limits
- QoS classes (Guaranteed, Burstable, BestEffort)
- LimitRanges
- ResourceQuotas
- Pod Priority and Preemption

**Key Labs:**

- Set resource constraints
- Create LimitRange
- Apply ResourceQuota
- Priority classes

### Day 13: Advanced Scheduling

**Topics:**

- Manual pod scheduling
- Node selectors
- Node/Pod affinity and anti-affinity
- Taints and tolerations
- Custom scheduler

**Key Labs:**

- Schedule pods to specific nodes
- Configure affinity rules
- Pod disruption budgets
- Topology spread constraints

### Day 14: Week 2 Review

**Practice Scenarios:**

- Deploy multi-tier application
- Implement rolling updates
- Configure resource quotas
- Set up scheduled jobs
- Advanced pod scheduling

---

## Week 3: Services, Networking & Storage (30% of exam)

### Day 15: Services Deep Dive

**Topics:**

- Service types (ClusterIP, NodePort, LoadBalancer)
- Service discovery
- Endpoints and EndpointSlices
- Headless services
- ExternalName services

**Key Labs:**

- Create various service types
- Service to pod communication
- External service access
- DNS resolution

### Day 16: Ingress Controllers

**Topics:**

- Ingress resources
- Ingress controllers (nginx)
- Path-based routing
- Host-based routing
- TLS termination

**Key Labs:**

- Deploy ingress controller
- Configure ingress rules
- SSL certificates
- Multiple host routing

### Day 17: Network Policies

**Topics:**

- Default deny policies
- Ingress rules
- Egress rules
- Pod/Namespace selectors
- Policy combinations

**Key Labs:**

- Deny all traffic
- Allow specific pods
- Egress restrictions
- Complex policy scenarios

### Day 18: DNS and CoreDNS

**Topics:**

- Kubernetes DNS
- CoreDNS configuration
- Service discovery via DNS
- Custom DNS entries
- DNS troubleshooting

**Key Labs:**

- Test DNS resolution
- Custom CoreDNS config
- Debugging DNS issues
- External DNS integration

### Day 19: Persistent Storage

**Topics:**

- PersistentVolumes (PV)
- PersistentVolumeClaims (PVC)
- StorageClasses
- Volume access modes
- Dynamic provisioning

**Key Labs:**

- Create PV and PVC
- Configure StorageClass
- Mount volumes in pods
- Expand volumes

### Day 20: Advanced Storage

**Topics:**

- Volume types (hostPath, emptyDir, configMap, secret)
- Volume snapshots
- StatefulSet storage
- CSI drivers

**Key Labs:**

- Use various volume types
- Backup with snapshots
- StatefulSet with PVCs
- Storage migration

### Day 21: Week 3 Review

**Practice Scenarios:**

- Deploy application with services
- Configure ingress routing
- Implement network policies
- Set up persistent storage
- End-to-end application deployment

---

## Week 4: Security & Troubleshooting (40% of exam)

### Day 22: Security Contexts and Pod Security

**Topics:**

- Security contexts (pod/container level)
- RunAsUser, RunAsGroup
- Capabilities
- SELinux/AppArmor
- Pod Security Standards

**Key Labs:**

- Configure security contexts
- Drop capabilities
- Read-only filesystems
- Enforce Pod Security Standards

### Day 23: Secrets and ConfigMaps

**Topics:**

- Creating secrets (generic, TLS, Docker)
- ConfigMaps for configuration
- Using secrets in pods
- Environment variables vs volumes
- Secret encryption at rest

**Key Labs:**

- Create and use secrets
- Mount ConfigMaps
- Environment variable injection
- Update config without restart

### Day 24: Network Security

**Topics:**

- Network policies (review)
- API server security
- TLS certificates
- Service meshes (concepts)
- Encryption in transit

**Key Labs:**

- Secure pod communication
- Certificate management
- Mutual TLS

### Day 25: Cluster Hardening

**Topics:**

- API authentication
- Admission controllers
- Audit logging
- Security benchmarks (CIS)
- Vulnerability scanning

**Key Labs:**

- Configure audit logs
- Enable admission controllers
- Review security settings

### Day 26: Troubleshooting Clusters

**Topics:**

- Node troubleshooting
- Control plane issues
- Network connectivity
- DNS problems
- Certificate issues

**Key Labs:**

- Fix broken nodes
- Debug API server
- Resolve DNS failures
- Certificate renewal

### Day 27: Troubleshooting Applications

**Topics:**

- Pod troubleshooting
- Container logs
- Debug containers
- Resource issues
- Image pull errors

**Key Labs:**

- Debug failing pods
- Inspect logs
- Ephemeral containers
- Fix resource constraints

### Day 28: Monitoring and Logging

**Topics:**

- kubectl top
- Resource metrics
- Log aggregation
- Events
- Health checks

**Key Labs:**

- Monitor resource usage
- Aggregate logs
- Alert on events
- Dashboard setup

### Day 29: Final Practice Exam

**Comprehensive Scenarios (3 hours):**

1. Cluster upgrade and backup
2. Deploy multi-tier application
3. Configure networking and security
4. Troubleshoot failing components
5. Implement RBAC policies
6. Storage configuration
7. Performance tuning

### Day 30: Exam Preparation

**Final Review:**

- Command cheatsheet
- Common pitfalls
- Time management strategies
- Exam environment tips
- Last-minute practice

**Key Commands to Memorize:**

```bash
# etcd backup
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Drain node
kubectl drain NODE --ignore-daemonsets --delete-emptydir-data

# Generate YAML
kubectl run NAME --image=IMAGE --dry-run=client -o yaml

# RBAC
kubectl create role NAME --verb=VERB --resource=RESOURCE -n NS
kubectl create rolebinding NAME --role=ROLE --serviceaccount=NS:SA -n NS

# Quick pod with overrides
kubectl run NAME --image=IMAGE --env="KEY=VALUE" --labels="key=value"
```

---

## Exam Tips

### Time Management

- **2 hours** for 15-20 questions
- ~6-8 minutes per question
- Flag difficult questions, return later
- Allocate extra time for cluster upgrades/backups

### Environment Setup

- Set alias: `alias k=kubectl`
- Enable bash completion
- Set namespace context to avoid -n flag
- Use `--dry-run=client -o yaml` extensively

### Common Mistakes to Avoid

1. Forgetting to verify your work
2. Not using dry-run before applying
3. Ignoring namespace requirements
4. Skipping documentation (kubectl explain)
5. Not practicing speed

### Documentation

- Kubernetes.io/docs is allowed
- Bookmark key pages:
  - kubectl cheat sheet
  - API reference
  - Tasks section
- Practice finding info quickly

### Verification Checklist

```bash
# Always verify:
kubectl get pods
kubectl describe pod NAME
kubectl logs POD
kubectl get events
kubectl auth can-i VERB RESOURCE
```

---

## Additional Resources

### Official Documentation

- <https://kubernetes.io/docs/home/>
- <https://kubernetes.io/docs/reference/kubectl/cheatsheet/>
- <https://kubernetes.io/docs/tasks/>

### Practice Environments

- killer.sh (2 free sessions with exam)
- katacoda.com/courses/kubernetes
- <https://github.com/dgkanatsios/CKAD-exercises>

### Communities

- Kubernetes Slack
- Reddit r/kubernetes
- Stack Overflow

### Books

- "Kubernetes: Up and Running"
- "Kubernetes in Action"

---

## Appendix: Quick Reference

### kubectl Shortcuts

```bash
k get po -A -o wide
k describe po NAME
k logs POD -c CONTAINER
k exec -it POD -- sh
k run NAME --image=IMAGE --dry-run=client -o yaml
k create deploy NAME --image=IMAGE --replicas=N
k expose deploy NAME --port=PORT --target-port=PORT
k scale deploy NAME --replicas=N
k set image deploy/NAME CONTAINER=IMAGE
k rollout status deploy/NAME
k rollout undo deploy/NAME
k drain NODE --ignore-daemonsets --delete-emptydir-data
k cordon NODE
k uncordon NODE
```

### Common YAML Templates

**Pod:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: NAME
spec:
  containers:
  - name: CONTAINER
    image: IMAGE
```

**Deployment:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: NAME
spec:
  replicas: N
  selector:
    matchLabels:
      app: LABEL
  template:
    metadata:
      labels:
        app: LABEL
    spec:
      containers:
      - name: CONTAINER
        image: IMAGE
```

**Service:**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: NAME
spec:
  selector:
    app: LABEL
  ports:
  - port: PORT
    targetPort: PORT
  type: ClusterIP
```

**Good luck with your CKA exam! 🚀**

Remember: Practice, practice, practice. The exam is entirely hands-on, so command-line proficiency is essential. Work through each day's content thoroughly, and you'll be well-prepared!
