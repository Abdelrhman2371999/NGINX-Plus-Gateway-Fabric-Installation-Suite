# Kubernetes Cluster Installation Guide Summary

This guide provides a comprehensive, step-by-step approach to setting up a 3-node Kubernetes cluster using Ubuntu VMs and kubeadm.

## 📋 **Cluster Architecture Overview**

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│   Master Node   │     │   Worker Node   │     │   Worker Node   │
│   k8s-master01  │◄────►   k8s-worker01  │◄────►   k8s-worker02  │
│  (172.20.101.77)│     │ (172.20.101.108)│     │ (172.20.101.79) │
└─────────────────┘     └─────────────────┘     └─────────────────┘
```

## 🚀 **Quick Installation Checklist**

### **Phase 1: SSH & Network Setup (All 3 VMs)**
- [ ] Install SSH: `sudo apt install openssh-server`
- [ ] Configure static IP in `/etc/netplan/01-netcfg.yaml`
- [ ] Disable swap: `sudo swapoff -a` & comment out in `/etc/fstab`
- [ ] Add hosts to `/etc/hosts` for name resolution

### **Phase 2: Firewall Configuration**
**Master Node Ports:** 6443, 2379:2380, 10250:10252, 10255, 30000:32767, 80, 443  
**Worker Node Ports:** 10250, 30000:32767

### **Phase 3: Kubernetes Components Installation (All 3 VMs)**
- [ ] Configure kernel modules & sysctl
- [ ] Install containerd (with systemd cgroup driver)
- [ ] Add Kubernetes repo & install kubeadm, kubelet, kubectl

### **Phase 4: Cluster Initialization (Master Only)**
- [ ] Pull images: `sudo kubeadm config images pull`
- [ ] Initialize: `sudo kubeadm init --pod-network-cidr=10.244.0.0/16`
- [ ] Configure kubectl for regular user
- [ ] Save the `kubeadm join` command output

### **Phase 5: Network & Worker Join**
- [ ] Deploy Calico CNI: `kubectl apply -f https://docs.tigera.io/calico/latest/manifests/calico.yaml`
- [ ] Join workers using saved `kubeadm join` command
- [ ] Verify: `kubectl get nodes` shows all nodes "Ready"

## 🔑 **Key Commands Reference**

| Purpose | Command |
|---------|---------|
| **Check cluster status** | `kubectl get nodes` |
| **Check all pods** | `kubectl get pods --all-namespaces` |
| **Test cluster connectivity** | `ping k8s-master01` from workers |
| **Verify container runtime** | `sudo systemctl status containerd` |
| **Check kubelet status** | `sudo systemctl status kubelet` |
| **View cluster join token** | `kubeadm token create --print-join-command` |

## ⚠️ **Critical Prerequisites**

1. **All VMs must:** Use Ubuntu (tested on 22.04/24.04), have unique hostnames, static IPs
2. **Network Requirements:** All nodes must be on same subnet, proper firewall rules
3. **Resource Requirements:** Minimum 2GB RAM, 2 vCPUs per node, 20GB storage
4. **User Permissions:** Need sudo/root access on all VMs

## 🛠️ **Troubleshooting Tips**

### **Common Issues:**
- **Nodes not joining:** Check token expiration (24h default), regenerate with `kubeadm token create`
- **Pods not starting:** Verify CNI plugin installed, check `kubectl describe pod <pod-name>`
- **Network connectivity issues:** Confirm firewall rules, check `ip a` for correct interface names
- **Container runtime issues:** Validate containerd config at `/etc/containerd/config.toml`

### **Verification Steps:**
```bash
# 1. Check all components are running
kubectl get componentstatus

# 2. Test pod creation
kubectl run test-nginx --image=nginx:alpine

# 3. Check cluster info
kubectl cluster-info
```

## 📝 **Post-Installation Recommendations**

1. **Backup kubeconfig:** Secure your `~/.kube/config` file
2. **Set up monitoring:** Consider Prometheus + Grafana for observability
3. **Implement backups:** Use Velero for cluster state backups
4. **Security hardening:** Apply RBAC, network policies, pod security standards
5. **Storage setup:** Configure StorageClass for dynamic provisioning

## 🔗 **Useful Resources**

- [Official kubeadm docs](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/)
- [Calico network policy docs](https://docs.tigera.io/calico/latest/)
- [Kubernetes the Hard Way](https://github.com/kelseyhightower/kubernetes-the-hard-way) for deeper understanding

---

**Estimated Time:** 30-60 minutes for experienced users, 2-3 hours for beginners  
**Difficulty:** Intermediate  
**Success Criteria:** All 3 nodes show "Ready" status in `kubectl get nodes`

