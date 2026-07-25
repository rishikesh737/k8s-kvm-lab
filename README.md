# Bare-Metal Kubernetes Cluster via KVM/QEMU

A 4-node Kubernetes cluster provisioned locally on Fedora 44 using KVM/QEMU, libvirt, and Rocky Linux 9.

## Architecture

```mermaid
graph TD
    subgraph Host["HP Z2 Tower G9 - Fedora 44"]
        KVM[KVM / QEMU Hypervisor]
        Bridge["libvirt default bridge<br/>192.168.122.0/24"]
        
        subgraph Cluster["Kubernetes v1.31.14 Cluster"]
            CP["k8s-control<br/>192.168.122.69"]
            W1["k8s-worker-1<br/>192.168.122.82"]
            W2["k8s-worker-2<br/>192.168.122.12"]
            W3["k8s-worker-3<br/>192.168.122.239"]
        end
        
        KVM --> Bridge
        Bridge --- CP
        Bridge --- W1
        Bridge --- W2
        Bridge --- W3
    end
```

## Stack

- **Hypervisor:** KVM/QEMU + libvirt
- **OS:** Rocky Linux 9.8 (GenericCloud image)
- **Container Runtime:** containerd 2.2.6
- **Orchestrator:** Kubernetes v1.31.14 (kubeadm)
- **CNI:** Flannel (VXLAN)

## Docs

- [Full setup guide](./k8s-kvm-setup-guide.md) — step by step provisioning, bootstrap, and troubleshooting
- [Known issues & fixes](./troubleshooting.md) — production-readiness gaps found post-deployment

## Quick Start

```bash
sudo virsh start k8s-control k8s-worker-1 k8s-worker-2 k8s-worker-3
sudo virsh net-dhcp-leases default
ssh admin@<control-plane-ip>
kubectl get nodes
```
