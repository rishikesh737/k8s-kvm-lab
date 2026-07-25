# KVM + Kubernetes Cluster Setup Guide
**Author:** Rishikesh Pednekar  
**Date:** July 13, 2026  
**System:** HP Z2 Tower G9 — Intel i9-12900 (24 threads), 31GB RAM, Fedora Linux 44  
**Goal:** Deploy 4 KVM VMs via CLI and bootstrap a Kubernetes cluster using kubeadm

---

## Table of Contents
1. [Architecture Overview](#architecture-overview)
2. [Phase 1: KVM Setup](#phase-1-kvm-setup)
3. [Phase 2: VM Provisioning](#phase-2-vm-provisioning)
4. [Phase 3: Kubernetes Prerequisites](#phase-3-kubernetes-prerequisites)
5. [Phase 4: Container Runtime](#phase-4-container-runtime)
6. [Phase 5: kubeadm Install](#phase-5-kubeadm-install)
7. [Phase 6: Cluster Bootstrap](#phase-6-cluster-bootstrap)
8. [Problems Encountered & Fixes](#problems-encountered--fixes)
9. [Tips to Make This Faster Next Time](#tips-to-make-this-faster-next-time)

---

## Architecture Overview

| VM | Role | vCPU | RAM | Disk | IP |
|---|---|---|---|---|---|
| k8s-control | Control Plane | 2 | 4GB | 20GB | 192.168.122.69 |
| k8s-worker-1 | Worker | 2 | 4GB | 20GB | 192.168.122.82 |
| k8s-worker-2 | Worker | 2 | 4GB | 20GB | 192.168.122.12 |
| k8s-worker-3 | Worker | 2 | 4GB | 20GB | 192.168.122.239 |

**Software Stack:**
- OS: Rocky Linux 9.8 (Blue Onyx) — GenericCloud image
- Container Runtime: containerd 2.2.6
- Kubernetes: v1.31.14
- CNI: Flannel (VXLAN)
- Hypervisor: KVM/QEMU via libvirt

---

## Phase 1: KVM Setup

### Verify KVM is available
```bash
egrep -c '(vmx|svm)' /proc/cpuinfo   # Should be > 0
lsmod | grep kvm
sudo virt-host-validate
```

### Install required tools (no GUI)
```bash
sudo dnf install -y \
  qemu-kvm \
  libvirt \
  libvirt-daemon-kvm \
  virt-install \
  libguestfs-tools \
  guestfs-tools \
  bridge-utils \
  virt-top \
  genisoimage
```

### Enable libvirt and add user to group
```bash
sudo systemctl enable --now libvirtd
sudo usermod -aG libvirt $USER
newgrp libvirt
```

---

## Phase 2: VM Provisioning

### Download Rocky Linux GenericCloud image
```bash
wget https://download.rockylinux.org/pub/rocky/9/images/x86_64/Rocky-9-GenericCloud.latest.x86_64.qcow2

# Move to libvirt images directory as base image
sudo mv ~/Rocky-9-GenericCloud.latest.x86_64.qcow2 /var/lib/libvirt/images/rocky9-base.qcow2
```

> **Why GenericCloud?** The network installer requires downloading 1.1GB into a tmpfs ramdisk during boot, which causes "No space left on device" errors. The GenericCloud image boots instantly with no installer needed.

### Create VM disks (copy-on-write from base)
```bash
for VM in k8s-control k8s-worker-1 k8s-worker-2 k8s-worker-3; do
  sudo qemu-img create -f qcow2 -F qcow2 \
    -b /var/lib/libvirt/images/rocky9-base.qcow2 \
    /var/lib/libvirt/images/${VM}.qcow2 20G
  echo "Created disk for ${VM}"
done
```

### Create cloud-init configuration
```bash
mkdir -p ~/cloud-init

cat > ~/cloud-init/user-data <<'EOF'
#cloud-config
users:
  - name: admin
    sudo: ALL=(ALL) NOPASSWD:ALL
    groups: wheel
    shell: /bin/bash
    lock_passwd: false
    passwd: $6$rounds=4096$saltsalt$<your-hash-here>

chpasswd:
  expire: false

ssh_pwauth: true
package_update: false
EOF
```

### Create cloud-init ISOs
```bash
for VM in k8s-control k8s-worker-1 k8s-worker-2 k8s-worker-3; do
  cat > ~/cloud-init/meta-data <<EOF
instance-id: ${VM}
local-hostname: ${VM}
EOF

  genisoimage -output ~/cloud-init/${VM}-cloud-init.iso \
    -volid cidata -rational-rock -joliet \
    ~/cloud-init/user-data ~/cloud-init/meta-data

  sudo mv ~/cloud-init/${VM}-cloud-init.iso /var/lib/libvirt/images/
  echo "Created and moved ISO for ${VM}"
done
```

### Boot all 4 VMs
```bash
for VM in k8s-control k8s-worker-1 k8s-worker-2 k8s-worker-3; do
  sudo virt-install \
    --name ${VM} \
    --ram 4096 \
    --vcpus 2 \
    --disk path=/var/lib/libvirt/images/${VM}.qcow2,format=qcow2 \
    --disk path=/var/lib/libvirt/images/${VM}-cloud-init.iso,device=cdrom \
    --os-variant rocky9 \
    --network network=default \
    --graphics none \
    --noautoconsole \
    --import
  echo "Started ${VM}"
done
```

### Check IPs and SSH in
```bash
sudo virsh net-dhcp-leases default
ssh admin@<IP>   # password set in cloud-init
```

### Useful virsh commands
```bash
sudo virsh list --all              # List all VMs
sudo virsh start <name>            # Start a VM
sudo virsh shutdown <name>         # Graceful shutdown
sudo virsh destroy <name>          # Force stop
sudo virsh undefine <name> --remove-all-storage  # Delete VM + disk
sudo virsh console <name>          # Attach to serial console
                                   # Escape: Ctrl + ]
```

---

## Phase 3: Kubernetes Prerequisites

Run on **all 4 VMs**:

```bash
# Disable swap (k8s requirement)
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab

# Load required kernel modules
sudo modprobe overlay
sudo modprobe br_netfilter

# Persist modules across reboots
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

# Set required sysctl params
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system

# Verify
sysctl net.ipv4.ip_forward   # Must show: 1
```

---

## Phase 4: Container Runtime

Run on **all 4 VMs**:

```bash
# Add Docker repo (contains containerd)
sudo dnf config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo

# Install containerd
sudo dnf install -y containerd.io

# Generate default config
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml

# Enable SystemdCgroup (required for Kubernetes)
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

# Enable and start
sudo systemctl enable --now containerd
sudo systemctl status containerd --no-pager
```

---

## Phase 5: kubeadm Install

Run on **all 4 VMs**:

```bash
# Add Kubernetes repo
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.31/rpm/
enabled=1
gpgcheck=0
exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
EOF

# Install
sudo dnf install -y kubelet kubeadm kubectl --disableexcludes=kubernetes

# Enable kubelet
sudo systemctl enable --now kubelet

# Also install conntrack (required by kubeadm preflight)
sudo dnf install -y conntrack-tools

# Verify
kubeadm version
kubectl version --client
```

---

## Phase 6: Cluster Bootstrap

### Initialize control plane (k8s-control only)
```bash
sudo kubeadm init \
  --pod-network-cidr=10.244.0.0/16 \
  --apiserver-advertise-address=192.168.122.69
```

### Set up kubectl access (k8s-control only)
```bash
mkdir -p $HOME/.kube
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### Install Flannel CNI (k8s-control only)
```bash
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

### Join worker nodes (on each worker)
```bash
# Use the join command printed by kubeadm init
sudo kubeadm join 192.168.122.69:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>
```

### Verify cluster (k8s-control only)
```bash
kubectl get nodes
kubectl get pods -n kube-system
kubectl get componentstatuses
```

**Expected output:**
```
NAME           STATUS   ROLES           AGE   VERSION
k8s-control    Ready    control-plane   17m   v1.31.14
k8s-worker-1   Ready    <none>          9m    v1.31.14
k8s-worker-2   Ready    <none>          9m    v1.31.14
k8s-worker-3   Ready    <none>          9m    v1.31.14
```

---

## Problems Encountered & Fixes

### 1. Network installer ran out of RAM
**Error:** `dracut: FATAL: Failed to find a root filesystem` / `No space left on device`  
**Cause:** The Anaconda network installer downloads 1.1GB into a tmpfs ramdisk. With only 2GB RAM allocated, it ran out of space.  
**Fix:** Switched to Rocky Linux GenericCloud image which boots directly without an installer. Much faster and more reliable.

---

### 2. cloud-init password hash didn't work
**Error:** `Login incorrect` when SSHing as admin  
**Cause:** The SHA-512 hash in the user-data was pre-generated for the literal string `yourpassword`, not `admin123`.  
**Fix:** Used `virt-customize` to reset passwords directly on the qcow2 disk images while VMs were shut down:
```bash
for VM in k8s-control k8s-worker-1 k8s-worker-2 k8s-worker-3; do
  sudo virt-customize \
    -a /var/lib/libvirt/images/${VM}.qcow2 \
    --root-password password:admin123 \
    --run-command 'echo "admin:admin123" | chpasswd' \
    --run-command 'echo "admin ALL=(ALL) NOPASSWD:ALL" > /etc/sudoers.d/admin' \
    --selinux-relabel
done
```

---

### 3. genisoimage permission denied writing to /var/lib/libvirt/images/
**Error:** `Permission denied. Unable to open disc image file`  
**Cause:** `/var/lib/libvirt/images/` requires root to write, but genisoimage was run as a regular user.  
**Fix:** Write ISOs to `~/cloud-init/` first, then `sudo mv` them to the libvirt directory.

---

### 4. containerd not found in Rocky Linux repos
**Error:** `No match for argument: containerd`  
**Cause:** containerd is not in the default Rocky Linux 9 repositories.  
**Fix:** Add the Docker CE repo which packages containerd:
```bash
sudo dnf config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo
sudo dnf install -y containerd.io
```

---

### 5. Kubernetes GPG key import failed
**Error:** `key 1 not an armored public key` / `GPG check FAILED`  
**Cause:** The `.asc` URL in the repo config points to repo metadata XML, not an actual GPG public key. The pkgs.k8s.io key format changed.  
**Fix:** Disabled GPG check for the Kubernetes repo (acceptable for a learning environment):
```bash
# In /etc/yum.repos.d/kubernetes.repo
gpgcheck=0
```

---

### 6. kubeadm preflight: conntrack not found
**Error:** `[ERROR FileExisting-conntrack]: conntrack not found in system path`  
**Cause:** `conntrack-tools` is not installed by default on Rocky Linux 9 GenericCloud.  
**Fix:**
```bash
sudo dnf install -y conntrack-tools
```
This needs to be installed on **all nodes** (control plane and workers) before running `kubeadm init` or `kubeadm join`.

---

### 7. Typo: `at` instead of `cat`
**Error:** `-bash: at: command not found` on k8s-worker-3  
**Cause:** Accidental typo when typing `cat <<EOF`. The sysctl config was never written so `ip_forward` stayed at 0.  
**Fix:** Re-ran the `cat <<EOF` block correctly on that node.

---

### 8. kubectl errors on worker nodes
**Error:** `dial tcp [::1]:8080: connect: connection refused`  
**Cause:** `kubectl` on worker nodes has no kubeconfig — workers are not API clients.  
**Fix:** This is expected and normal. Only run `kubectl` on the control plane node where `~/.kube/config` exists.

---

## Tips to Make This Faster Next Time

### 1. Pre-generate a correct password hash
Don't use a placeholder hash. Generate it before writing user-data:
```bash
python3 -c "import crypt; print(crypt.crypt('yourpassword', crypt.mksalt(crypt.METHOD_SHA512)))"
```
Paste the output directly into `user-data` — test it first by logging in before creating all VMs.

### 2. Use a pre-baked base image
Instead of configuring each VM after boot, use `virt-customize` on the base image once before cloning:
```bash
sudo virt-customize -a /var/lib/libvirt/images/rocky9-base.qcow2 \
  --install conntrack-tools \
  --run-command 'dnf config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo' \
  --install containerd.io \
  --selinux-relabel
```
Then every cloned VM already has all dependencies — no per-VM installs needed.

### 3. Use a script for the full k8s prep
Save this as `k8s-node-prep.sh` and run it on all nodes at once:
```bash
#!/bin/bash
set -e

# Swap
swapoff -a
sed -i '/ swap / s/^/#/' /etc/fstab

# Modules
modprobe overlay && modprobe br_netfilter
cat <<EOF > /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

# Sysctl
cat <<EOF > /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
sysctl --system

# containerd
dnf config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo -y
dnf install -y containerd.io conntrack-tools
mkdir -p /etc/containerd
containerd config default > /etc/containerd/config.toml
sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
systemctl enable --now containerd

# Kubernetes
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.31/rpm/
enabled=1
gpgcheck=0
exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
EOF
dnf install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
systemctl enable --now kubelet

echo "Node prep complete!"
```

Copy and run it on all nodes via a loop from the host:
```bash
for IP in 192.168.122.69 192.168.122.82 192.168.122.12 192.168.122.239; do
  scp k8s-node-prep.sh admin@${IP}:~/
  ssh admin@${IP} "sudo bash ~/k8s-node-prep.sh"
done
```

### 4. Set static IPs via cloud-init
DHCP can give different IPs after a reboot. Add this to your `user-data` to fix IPs:
```yaml
network:
  version: 2
  ethernets:
    enp1s0:
      addresses: [192.168.122.10/24]
      gateway4: 192.168.122.1
      nameservers:
        addresses: [8.8.8.8]
```

### 5. Use SSH keys instead of passwords
Much more secure and no password prompts. Add to `user-data`:
```yaml
ssh_authorized_keys:
  - ssh-ed25519 AAAA... your-public-key
```
Generate with: `ssh-keygen -t ed25519`

### 6. Install conntrack-tools in the base image prep
Since it's always needed for kubeadm, just add it to the initial VM setup or the baked base image so you never hit that preflight error again.

---

## Final Cluster State

```
NAME           STATUS   ROLES           VERSION
k8s-control    Ready    control-plane   v1.31.14
k8s-worker-1   Ready    <none>          v1.31.14
k8s-worker-2   Ready    <none>          v1.31.14
k8s-worker-3   Ready    <none>          v1.31.14

controller-manager   Healthy
scheduler            Healthy
etcd-0               Healthy
```

**Pod CIDRs assigned per node:**
- k8s-control:  `10.244.0.0/24`
- k8s-worker-1: `10.244.1.0/24`
- k8s-worker-2: `10.244.2.0/24`
- k8s-worker-3: `10.244.3.0/24`

---

## What's Next

```bash
# Deploy your first workload
kubectl create deployment nginx --image=nginx
kubectl expose deployment nginx --port=80 --type=NodePort
kubectl get pods -o wide      # See which node it landed on
kubectl get svc nginx         # Get the NodePort

# Scale it
kubectl scale deployment nginx --replicas=3
kubectl get pods -o wide      # Now spread across workers

# Clean up
kubectl delete deployment nginx
kubectl delete svc nginx
```
