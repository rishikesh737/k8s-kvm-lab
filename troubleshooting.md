# Post-Deployment Issues & Fixes

Issues discovered after the cluster had been running for ~12 days, found via systematic health-check auditing rather than at initial bootstrap.

---

## 1. Cross-Node Pod Network Failure (Flannel VXLAN blocked by firewalld)

**Symptom:**
```
kubectl run -it --image=alpine test-pod -- sh
/ # ping 10.244.1.2
100% packet loss
```

**Root cause:**
Rocky Linux 9 ships with `firewalld` enabled by default. Flannel's VXLAN backend requires **UDP 8472** open between every node for pod-to-pod traffic that crosses node boundaries. Same-node pod traffic works fine (it never leaves the host), which is why this can go unnoticed until you specifically test cross-node connectivity.

**Fix (run on all 4 nodes):**
```bash
sudo firewall-cmd --permanent --add-port=8472/udp
sudo firewall-cmd --permanent --add-port=6443/tcp   # apiserver
sudo firewall-cmd --permanent --add-port=10250/tcp  # kubelet
sudo firewall-cmd --reload
```

**Verification:**
```bash
kubectl run -it --image=alpine test1 --overrides='{"spec":{"nodeSelector":{"kubernetes.io/hostname":"k8s-worker-1"}}}' -- sh
# note the pod IP, exit
kubectl run -it --image=alpine test2 --overrides='{"spec":{"nodeSelector":{"kubernetes.io/hostname":"k8s-worker-2"}}}' -- sh
# ping the first pod's IP — should now succeed
```

**Why this matters:** any multi-node CNI (Flannel, Calico, Cilium) has a required port set that has to be opened on the host firewall. This is one of the most common "why can't my pods talk to each other" issues in real RHEL/Rocky clusters, since RHEL-family distros enable firewalld by default (unlike Ubuntu's iptables/ufw being mostly open).

---

## 2. Kubelet `InvalidDiskCapacity: invalid capacity 0 on image filesystem`

**Symptom:**
```
kubectl describe node k8s-worker-1
...
Warning  InvalidDiskCapacity   invalid capacity 0 on image filesystem
```

**Root cause:**
containerd's default generated config doesn't always correctly report the underlying filesystem stats to kubelet's `ImageFs` stats provider immediately after a fresh `containerd config default` write. This breaks:
- Image garbage collection (kubelet won't reliably prune unused images)
- Disk-pressure eviction (kubelet can't tell when the node is low on image storage)

**Fix (run on all 4 nodes):**
```bash
sudo systemctl restart containerd
sudo systemctl restart kubelet
```

In most cases the stats populate correctly after both services are cold-restarted together (containerd first, then kubelet re-queries it). If it persists, explicitly verify the snapshotter root path exists and is writable:
```bash
grep -A3 "snapshotter.v1.overlayfs" /etc/containerd/config.toml
ls -la /var/lib/containerd/io.containerd.snapshotter.v1.overlayfs
```

**Verification:**
```bash
kubectl describe node k8s-worker-1 | grep -A5 Conditions
# InvalidDiskCapacity warning should no longer appear in Events
```

---

## Lesson for Production Clusters

Both of these issues are **silent** — the cluster reports `Ready` and workloads schedule fine until you specifically stress cross-node networking or disk accounting. This is why health-check auditing (not just "is it Ready") matters:

```bash
# Don't just check this:
kubectl get nodes

# Also check:
kubectl run test --image=alpine --rm -it -- ping <pod-ip-on-different-node>
kubectl describe nodes | grep -A2 Conditions
sudo firewall-cmd --list-all   # on every node
```
