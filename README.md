# Bare-Metal Kubernetes Cluster via KVM/QEMU

This repository contains the documentation and architecture for a 4-node Kubernetes cluster provisioned locally on Fedora 44 using KVM/QEMU, `libvirt`, and Rocky Linux 9.

## Architecture Diagram

```mermaid
graph TD
    subgraph Host[HP Z2 Tower G9 - Fedora 44]
        KVM[KVM / QEMU Hypervisor]
        Bridge[libvirt default bridge: 192.168.122.0/24]
        
        subgraph Cluster[Kubernetes v1.31.14 Cluster]
            CP[k8s-control<br>192.168.122.69]
            W1[k8s-worker-1<br>192.168.122.82]
            W2[k8s-worker-2<br>192.168.122.12]
            W3[k8s-worker-3<br>192.168.122.239]
            
            CP --- Bridge
            W1 --- Bridge
            W2 --- Bridge
            W3 --- Bridge
        end
        
        KVM --> Cluster
    end
