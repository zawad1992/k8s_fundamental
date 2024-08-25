
# Common Kubernetes Flannel Pod Network CIDR Issues

When setting up Kubernetes, beginners often encounter issues related to the Pod Network CIDR configuration, especially when using Flannel as the CNI (Container Network Interface) plugin. Below is a brief explanation of the common issue and how to resolve it.

## Common Issue: PodCIDR Mismatch

### Symptom:
- The `kube-flannel` pods may fail to start with an error similar to:

```
Error registering network: failed to acquire lease: subnet "10.244.0.0/16" specified in the flannel net config doesn't contain "192.168.1.0/24" PodCIDR of the "worker1" node
```

### Cause:
- This error occurs due to a mismatch between the Pod Network CIDR specified during the Kubernetes cluster initialization and the network CIDR configured in Flannel.

### Example:
- The Kubernetes cluster was initialized with:
  ```
  kubeadm init --pod-network-cidr=192.168.0.0/16
  ```
- However, Flannel's default configuration uses the `10.244.0.0/16` network range.

## Solutions:

### Option 1: Modify the Flannel Configuration
1. Edit the Flannel ConfigMap:
   ```bash
   kubectl edit configmap -n kube-flannel kube-flannel-cfg
   ```
2. Change the `Network` value in the `net-conf.json` section to match the PodCIDR:
   ```json
   "Network": "192.168.0.0/16",
   ```
3. Restart the Flannel DaemonSet:
   ```bash
   kubectl rollout restart daemonset/kube-flannel-ds -n kube-flannel
   ```

### Option 2: Reinitialize the Cluster with the Correct Pod Network CIDR
1. Reset the Cluster:
   ```bash
   sudo kubeadm reset
   ```
2. Reinitialize with the correct Pod Network CIDR:
   ```bash
   sudo kubeadm init --pod-network-cidr=10.244.0.0/16
   ```
3. Reapply the Flannel Network Plugin:
   ```bash
   kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
   ```

## Conclusion:
Ensuring that the Pod Network CIDR specified during cluster initialization matches the CNI plugin's configuration is crucial. This brief guide should help resolve the common PodCIDR mismatch issue when using Flannel with Kubernetes.

---

*For more information, visit the official [Flannel documentation](https://github.com/coreos/flannel).*
