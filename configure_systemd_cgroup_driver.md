
# Configuring systemd Cgroup Driver in containerd with cgroup v2

## Introduction

When running Kubernetes with containerd as the container runtime and cgroup v2 as the cgroup manager, it is recommended to use the `systemd` cgroup driver. This configuration ensures that resources are managed correctly and aligns with the systemd initialization system, which is commonly used in modern Linux distributions.

## Problem

If you do not configure the `systemd` cgroup driver in `/etc/containerd/config.toml` when using cgroup v2, you might encounter the following issues:

1. **Pod and Container Instability**: Containers and pods may fail to start or experience instability because the default cgroup driver may not manage resources properly under cgroup v2.
2. **Compatibility Issues**: Kubernetes may face compatibility issues with resource management, leading to errors in pod scheduling or resource assignment.
3. **Errors in Kubelet Logs**: The Kubelet may log errors related to cgroup management, such as:
   ```
   Failed to start ContainerManager failed to initialize top level QOS containers: [failed to find container: "/kubepods"]
   ```
4. **Resource Monitoring Problems**: Tools that rely on cgroups for monitoring resource usage might not work correctly.
5. **Pod Evictions**: You may experience unexpected pod evictions due to improper resource accounting by the Kubelet.

## Solution

To resolve these issues, you need to configure containerd to use the systemd cgroup driver. Here's how to do it:

### Step 1: Edit containerd configuration

Open the containerd configuration file located at `/etc/containerd/config.toml` in a text editor:

```bash
sudo nano /etc/containerd/config.toml
```

### Step 2: Configure systemd cgroup driver

Find the section for the `runc` runtime and add the `SystemdCgroup` option as shown below:

```toml
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
  ...
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
    SystemdCgroup = true
```

### Step 3: Restart containerd

After making the changes, restart the containerd service to apply the new configuration:

```bash
sudo systemctl restart containerd
```

### Step 4: Verify the configuration

To verify that the systemd cgroup driver is being used, you can check the cgroup driver of a running container:

```bash
docker info | grep Cgroup
```

Ensure that it shows `systemd` as the cgroup driver.

## Conclusion

By configuring containerd to use the systemd cgroup driver with cgroup v2, you can avoid the issues mentioned above and ensure stable operation of your Kubernetes cluster.
