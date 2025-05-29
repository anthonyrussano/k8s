# Kubernetes Cluster Recovery Guide

## Overview
This guide documents the steps to recover a Kubernetes cluster after an unexpected power outage or node restart. The primary issue encountered was network connectivity failure due to missing kernel modules required by the Flannel CNI plugin.

## Root Cause Analysis
The power outage caused all nodes to restart, and the `br_netfilter` kernel module was not automatically loaded on some nodes during boot. This module is essential for Flannel's VXLAN backend networking functionality.

## Recovery Steps

### Step 1: Assess Cluster State
First, check the overall health of your cluster:

```bash
# Check node status
kubectl get nodes

# Check all pods across namespaces
kubectl get pods --all-namespaces

# Look for pods in problematic states (CrashLoopBackOff, Unknown, Error)
```

### Step 2: Identify Networking Issues
Look specifically for CNI plugin problems:

```bash
# Check Flannel pods status
kubectl get pods -n kube-flannel

# Check logs of failing Flannel pods
kubectl logs -n kube-flannel <pod-name>
```

**Common Error Pattern:** `Failed to check br_netfilter: stat /proc/sys/net/bridge/bridge-nf-call-iptables: no such file or directory`

### Step 3: Load Required Kernel Module
Create a privileged DaemonSet to load the `br_netfilter` module on all nodes:

```bash
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: load-br-netfilter
  namespace: kube-system
  labels:
    app: load-br-netfilter
spec:
  selector:
    matchLabels:
      app: load-br-netfilter
  template:
    metadata:
      labels:
        app: load-br-netfilter
    spec:
      hostPID: true
      hostNetwork: true
      containers:
      - name: load-module
        image: busybox
        command: ['sh', '-c', 'nsenter -t 1 -m -u -i -n -p -- modprobe br_netfilter && echo "Module loaded" || echo "Failed to load module"; sleep 3600']
        securityContext:
          privileged: true
      tolerations:
      - effect: NoSchedule
        operator: Exists
      - effect: NoExecute
        operator: Exists
EOF
```

### Step 4: Verify Kernel Module Loading
Check that the DaemonSet pods are running and the module was loaded:

```bash
# Check DaemonSet pod status
kubectl get pods -n kube-system -l app=load-br-netfilter

# Check logs to confirm module loading
kubectl logs -n kube-system <load-br-netfilter-pod-name>
```

Expected output: `Module loaded`

### Step 5: Restart Flannel Pods
Force recreation of Flannel pods now that the kernel module is available:

```bash
# Delete existing Flannel pods to trigger recreation
kubectl delete pod -n kube-flannel -l app=flannel

# Verify new pods are running
kubectl get pods -n kube-flannel
```

### Step 6: Verify Cluster Recovery
Check that all services are back online:

```bash
# Check all pods status
kubectl get pods --all-namespaces

# Restart any remaining problematic pods
kubectl delete pod -n <namespace> <pod-name>
```

### Step 7: Clean Up (Optional)
Remove the temporary DaemonSet after confirming cluster stability:

```bash
kubectl delete daemonset -n kube-system load-br-netfilter
```

## Alternative Manual Method
If you have direct SSH access to nodes, you can manually load the kernel module:

```bash
# On each node, check if module is loaded
lsmod | grep br_netfilter

# Load the module if missing
sudo modprobe br_netfilter

# Verify it's loaded
lsmod | grep br_netfilter

# Make it persistent across reboots
echo 'br_netfilter' | sudo tee -a /etc/modules-load.d/k8s.conf
```

## Prevention Strategies

### 1. Persistent Kernel Module Loading
Ensure required kernel modules are loaded automatically on boot by adding them to system configuration:

```bash
# On each node, create or edit the modules configuration
sudo tee /etc/modules-load.d/k8s.conf <<EOF
br_netfilter
overlay
EOF

# Also ensure sysctl settings persist
sudo tee /etc/sysctl.d/k8s.conf <<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF

# Apply sysctl settings
sudo sysctl --system
```

### 2. Node Configuration Management
Use configuration management tools (Ansible, Puppet, etc.) to ensure consistent node setup across all cluster nodes.

### 3. Monitoring and Alerting
Implement monitoring for:
- Node readiness status
- CNI plugin pod health
- Network connectivity between nodes
- Kernel module availability

### 4. Graceful Shutdown Procedures
Implement proper shutdown procedures during planned maintenance:

```bash
# Drain nodes before shutdown
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data

# Uncordon after restart
kubectl uncordon <node-name>
```

### 5. CNI Plugin Alternatives
Consider CNI plugins with better resilience to kernel module issues:
- **Calico**: More robust networking with advanced features
- **Cilium**: eBPF-based networking with better observability
- **Weave**: Simple overlay network with automatic configuration

### 6. Cluster Backup Strategy
Regular backups of:
- etcd data
- Cluster configuration
- Application manifests
- Persistent volume data

### 7. Infrastructure Improvements
- **UPS Systems**: Prevent unexpected shutdowns
- **Node Auto-recovery**: Implement scripts to automatically configure nodes on boot
- **Health Checks**: Regular automated cluster health validation

## Troubleshooting Tips

### Common Issues After Recovery
1. **Persistent Volumes not mounting**: Check storage class and PV status
2. **DNS resolution failures**: Restart CoreDNS pods if needed
3. **Ingress controller issues**: Verify ingress controller pods are ready
4. **Certificate issues**: Check if certificates expired during downtime

### Useful Commands
```bash
# Check cluster events
kubectl get events --sort-by=.metadata.creationTimestamp

# Check node conditions
kubectl describe nodes

# Check system pod logs
kubectl logs -n kube-system <pod-name>

# Force delete stuck pods
kubectl delete pod <pod-name> --grace-period=0 --force
```

## Summary
The key to recovery was identifying that the `br_netfilter` kernel module was missing after the power outage, which prevented the Flannel CNI plugin from functioning. By using a privileged DaemonSet to load the module across all nodes and then restarting the affected pods, the entire cluster was successfully restored.

Prevention focuses on ensuring kernel modules are persistently configured and implementing robust monitoring and infrastructure practices to minimize the impact of future outages.

## Key Recommendations for Future Resilience

The most important thing you can do is **make the kernel module loading persistent**. The recovery was successful but temporary - without persistence, you'll face the same issue after the next reboot.

**High Priority Actions:**
1. **SSH into each node** and run:
   ```bash
   echo 'br_netfilter' | sudo tee -a /etc/modules-load.d/k8s.conf
   ```
2. **Set up monitoring** for CNI pod health
3. **Consider switching CNI plugins** - Calico or Cilium are generally more robust than Flannel

**Infrastructure Improvements:**
- Install UPS systems to prevent sudden power loss
- Implement node configuration management (Ansible/Puppet)
- Set up cluster health monitoring and alerting
- Create automated backup procedures for etcd and configurations
