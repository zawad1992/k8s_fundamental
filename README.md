### Kubernetes Cluster Reset and Reinstallation Documentation

This documentation provides step-by-step instructions for resetting an existing Kubernetes cluster, uninstalling the necessary packages, and reinstalling Kubernetes with Containerd as the container runtime. The instructions are tailored for CentOS-based systems.

---

### 1. Reset Kubernetes Cluster

This section reverts the changes made by `kubeadm init` or `kubeadm join`.

```bash
sudo kubeadm reset
```

### 2. Remove Kubernetes Packages

Use the `yum` command to uninstall Kubernetes packages.

```bash
sudo yum remove -y kubeadm kubectl kubelet kubernetes-cni kube*
```

### 3. Clean Up Remaining Dependencies

Remove any remaining dependencies that were installed as part of Kubernetes.

```bash
sudo yum autoremove -y
```

### 4. Remove Kubernetes Configuration and Data Directories

Delete Kubernetes configuration and data directories to ensure a clean environment.

```bash
sudo rm -rf ~/.kube
sudo rm -rf /etc/kubernetes /var/lib/kubelet /var/lib/etcd /etc/cni
sudo rm -f /etc/apparmor.d/docker /etc/systemd/system/etcd*
sudo rm -rf /var/lib/dockershim /var/run/kubernetes
```

### 5. Flush IPTables

Clear any remaining network rules related to Kubernetes.

```bash
sudo iptables --flush
sudo iptables -F && iptables -X
sudo iptables -t nat -F && iptables -t nat -X
sudo iptables -t raw -F && iptables -t raw -X
sudo iptables -t mangle -F && iptables -t mangle -X
```

---

### 6. Configure sysctl for Kubernetes

Configure sysctl settings to ensure proper networking for Kubernetes.

```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF
```

Apply the sysctl settings:

```bash
sudo sysctl --system
```

---

### 7. Uninstall Docker (Optional)

If you previously used Docker, uninstall it to switch to Containerd as the CRI runtime.

```bash
sudo yum remove -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
sudo yum remove -y docker docker-client docker-client-latest docker-common docker-latest docker-latest-logrotate docker-logrotate docker-engine
```

---

### 8. Install Containerd

Follow these steps to install and configure Containerd as your CRI runtime.

1. **Install Containerd**:
    - Install Containerd using the official instructions.
  
2. **Configure Containerd**:
    - Create or update the `config.toml` file located at `/etc/containerd/config.toml`.
    - Ensure that the systemd cgroup driver is set by adding the following configuration:

    ```toml
    [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
      ...
      [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
        SystemdCgroup = true
    ```

3. **Start Containerd**:
    ```bash
    sudo systemctl enable --now containerd
    ```

### 9. Disable Swap and Set SELinux to Permissive Mode

1. **Disable Swap**:
    ```bash
    sudo swapoff -a
    sudo mount -a
    ```

2. **Set SELinux to Permissive**:
    ```bash
    sudo setenforce 0
    sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
    ```

---

### 10. Add Kubernetes Yum Repository

Add the Kubernetes repository for the specific version you want to install (Kubernetes 1.30 in this example).

```bash
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.30/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.30/rpm/repodata/repomd.xml.key
exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
EOF
```

### 11. Install Kubernetes Components

Install `kubelet`, `kubeadm`, and `kubectl` using the yum package manager.

```bash
sudo yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
```

Enable the `kubelet` service to start automatically on system boot.

```bash
sudo systemctl enable --now kubelet
```

---

### 12. Initialize the Kubernetes Master Node

Use the `kubeadm init` command to initialize the control plane. Replace `<your-master-node-ip>` with your actual IP address.

```bash
sudo kubeadm init --apiserver-advertise-address=<your-master-node-ip> --pod-network-cidr=192.168.0.0/16 --cri-socket=unix:///var/run/containerd/containerd.sock
```

If you use a different Pod network CIDR, adjust the `--pod-network-cidr` accordingly.

---

### 13. Configure kubectl for the Regular User

After the control plane is initialized, set up the `kubectl` configuration for your user.

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Alternatively, if you're using the root user, you can set the `KUBECONFIG` environment variable:

```bash
export KUBECONFIG=/etc/kubernetes/admin.conf
```

---

### 14. Install a Network Plugin

Install a network plugin like Flannel or Calico to set up the pod network.

**Flannel:**

```bash
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

**Calico:**

```bash
curl https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/calico.yaml -O
kubectl apply -f calico.yaml
```

Monitor the network plugin pods:

```bash
watch kubectl get pods -n calico-system
```

---

### 15. Allow Scheduling on the Control Plane Node (Optional)

If you want to schedule pods on the control plane node, remove the taints:

```bash
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```

---

### 16. Join Worker Nodes to the Cluster

On each worker node, ensure port `6443` is open on the master node:

```bash
sudo firewall-cmd --list-all | grep 6443
```

If necessary, open the port:

```bash
sudo firewall-cmd --permanent --add-port=6443/tcp
sudo firewall-cmd --reload
```

Generate a join command on the master node:

```bash
kubeadm token create --print-join-command
```

Run the join command on each worker node.

---

### 17. Verify Node Status

Check the status of your nodes:

```bash
kubectl get nodes -o wide
```

Congratulations! You now have a functional Kubernetes cluster with Containerd as the container runtime.

---
### 18. Troubleshoot

If you encounter any issues during the installation or operation of your Kubernetes cluster, refer to the following troubleshooting guides:

- [Network Issues with Flannel in Kubernetes](link_to_flannel_network_issues.md)

Ensure that your system's configurations align with the documentation, and review the Kubernetes logs for detailed error messages.

---
