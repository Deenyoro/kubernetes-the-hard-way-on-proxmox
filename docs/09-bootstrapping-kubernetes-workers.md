Below is the revised guide for bootstrapping your Kubernetes worker nodes in your Pittsburgh, PA environment. In your setup, the worker nodes are named as follows:

- **worker-214** with IP: 192.168.1.214  
- **worker-241** with IP: 192.168.1.241  
- **worker-242** with IP: 192.168.1.242  
- **worker-243** with IP: 192.168.1.243  
- **worker-244** with IP: 192.168.1.244

Each worker node will run the following components: [runc](https://github.com/opencontainers/runc), [container networking plugins](https://github.com/containernetworking/cni), [containerd](https://github.com/containerd/containerd), [kubelet](https://kubernetes.io/docs/admin/kubelet), and [kube-proxy](https://kubernetes.io/docs/concepts/cluster-administration/proxies).

The instructions below assume you’ll set a unique Pod CIDR for each worker. For example, you might assign:
- **worker-214**: 10.200.0.0/24  
- **worker-241**: 10.200.1.0/24  
- **worker-242**: 10.200.2.0/24  
- **worker-243**: 10.200.3.0/24  
- **worker-244**: 10.200.4.0/24

Follow these steps on each worker node (log in via SSH, e.g., `ssh root@worker-214`):

---

# Bootstrapping the Kubernetes Worker Nodes

In this lab you will bootstrap the Kubernetes worker nodes. Each worker node in our Pittsburgh, PA environment will run runc, container networking plugins, containerd, kubelet, and kube-proxy.

## Prerequisites

Run these commands on each worker node: **worker-214**, **worker-241**, **worker-242**, **worker-243**, and **worker-244**. For example:

```bash
ssh root@worker-214
```

### Running Commands in Parallel with tmux

[tmux](https://github.com/tmux/tmux/wiki) can be used to run commands simultaneously on multiple nodes. See the [Running commands in parallel with tmux](01-prerequisites.md#running-commands-in-parallel-with-tmux) section in the Prerequisites lab.

## Provisioning a Kubernetes Worker Node

### 1. Install OS Dependencies

Update packages and install required utilities:

```bash
sudo apt-get update
sudo apt-get -y install socat conntrack ipset
```

> The socat binary enables support for the `kubectl port-forward` command.

### 2. Disable Swap

Kubelet requires swap to be disabled. Check swap status:

```bash
sudo swapon --show
```

If swap is enabled, disable it immediately:

```bash
sudo swapoff -a
```

> To disable swap permanently, comment out the swap entry in `/etc/fstab` as needed.

### 3. Download and Install Worker Binaries

Download the necessary binaries:

```bash
wget -q --show-progress --https-only --timestamping \
  https://github.com/kubernetes-sigs/cri-tools/releases/download/v1.29.0/crictl-v1.29.0-linux-amd64.tar.gz \
  https://github.com/opencontainers/runc/releases/download/v1.1.12/runc.amd64 \
  https://github.com/containernetworking/plugins/releases/download/v1.4.0/cni-plugins-linux-amd64-v1.4.0.tgz \
  https://github.com/containerd/containerd/releases/download/v1.7.13/containerd-1.7.13-linux-amd64.tar.gz \
  https://storage.googleapis.com/kubernetes-release/release/v1.29.1/bin/linux/amd64/kubectl \
  https://storage.googleapis.com/kubernetes-release/release/v1.29.1/bin/linux/amd64/kube-proxy \
  https://storage.googleapis.com/kubernetes-release/release/v1.29.1/bin/linux/amd64/kubelet
```

Create the installation directories:

```bash
sudo mkdir -p \
  /etc/cni/net.d \
  /opt/cni/bin \
  /var/lib/kubelet \
  /var/lib/kube-proxy \
  /var/lib/kubernetes \
  /var/run/kubernetes
```

Install the binaries:

```bash
mkdir containerd
tar -xvf crictl-v1.29.0-linux-amd64.tar.gz
tar -xvf containerd-1.7.13-linux-amd64.tar.gz -C containerd
sudo tar -xvf cni-plugins-linux-amd64-v1.4.0.tgz -C /opt/cni/bin/
sudo mv runc.amd64 runc
chmod +x crictl kubectl kube-proxy kubelet runc
sudo mv crictl kubectl kube-proxy kubelet runc /usr/local/bin/
sudo mv containerd/bin/* /bin/
```

### 4. Configure CNI Networking

Define the Pod CIDR for the current node. Replace `THE_POD_CIDR` with the appropriate CIDR. For example, on **worker-214**:

```bash
POD_CIDR=10.200.0.0/24
```

> On other nodes, assign a unique CIDR (e.g., worker-241: 10.200.1.0/24, etc.).

Create the bridge network configuration:

```bash
cat <<EOF | sudo tee /etc/cni/net.d/10-bridge.conf
{
    "cniVersion": "1.0.0",
    "name": "bridge",
    "type": "bridge",
    "bridge": "cnio0",
    "isGateway": true,
    "ipMasq": true,
    "ipam": {
        "type": "host-local",
        "ranges": [
          [{"subnet": "${POD_CIDR}"}]
        ],
        "routes": [{"dst": "0.0.0.0/0"}]
    }
}
EOF
```

Create the loopback network configuration:

```bash
cat <<EOF | sudo tee /etc/cni/net.d/99-loopback.conf
{
    "cniVersion": "1.0.0",
    "name": "lo",
    "type": "loopback"
}
EOF
```

### 5. Configure containerd

Create the containerd configuration directory:

```bash
sudo mkdir -p /etc/containerd/
```

Generate the default containerd configuration with systemd Cgroups enabled:

```bash
containerd config default | sed 's/SystemdCgroup = false/SystemdCgroup = true/' | sudo tee /etc/containerd/config.toml
```

Create the containerd systemd service unit:

```bash
cat <<EOF | sudo tee /etc/systemd/system/containerd.service
[Unit]
Description=containerd container runtime
Documentation=https://containerd.io
After=network.target

[Service]
ExecStartPre=/sbin/modprobe overlay
ExecStart=/bin/containerd
Restart=always
RestartSec=5
Delegate=yes
KillMode=process
OOMScoreAdjust=-999
LimitNOFILE=1048576
LimitNPROC=infinity
LimitCORE=infinity

[Install]
WantedBy=multi-user.target
EOF
```

### 6. Configure the Kubelet

Assuming you have already copied the worker node’s certificates and kubeconfig file to the node (with filenames matching the hostname, e.g. **worker-214.pem**, **worker-214-key.pem**, and **worker-214.kubeconfig**), move them into place:

```bash
sudo mv ${HOSTNAME}-key.pem ${HOSTNAME}.pem /var/lib/kubelet/
sudo mv ${HOSTNAME}.kubeconfig /var/lib/kubelet/kubeconfig
sudo mv ca.pem /var/lib/kubernetes/
```

Create the kubelet configuration file:

```bash
cat <<EOF | sudo tee /var/lib/kubelet/kubelet-config.yaml
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
authentication:
  anonymous:
    enabled: false
  webhook:
    enabled: true
  x509:
    clientCAFile: "/var/lib/kubernetes/ca.pem"
authorization:
  mode: Webhook
cgroupDriver: systemd
clusterDomain: "cluster.local"
clusterDNS:
  - "10.32.0.10"
podCIDR: "${POD_CIDR}"
resolvConf: "/run/systemd/resolve/resolv.conf"
runtimeRequestTimeout: "15m"
tlsCertFile: "/var/lib/kubelet/\${HOSTNAME}.pem"
tlsPrivateKeyFile: "/var/lib/kubelet/\${HOSTNAME}-key.pem"
EOF
```

Create the kubelet systemd service unit:

```bash
cat <<EOF | sudo tee /etc/systemd/system/kubelet.service
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/kubernetes/kubernetes
After=containerd.service
Requires=containerd.service

[Service]
ExecStart=/usr/local/bin/kubelet \\
  --config=/var/lib/kubelet/kubelet-config.yaml \\
  --kubeconfig=/var/lib/kubelet/kubeconfig \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### 7. Configure the Kubernetes Proxy

Move the kube-proxy kubeconfig into place:

```bash
sudo mv kube-proxy.kubeconfig /var/lib/kube-proxy/kubeconfig
```

Create the kube-proxy configuration file:

```bash
cat <<EOF | sudo tee /var/lib/kube-proxy/kube-proxy-config.yaml
kind: KubeProxyConfiguration
apiVersion: kubeproxy.config.k8s.io/v1alpha1
clientConnection:
  kubeconfig: "/var/lib/kube-proxy/kubeconfig"
mode: "iptables"
clusterCIDR: "10.200.0.0/16"
EOF
```

Create the kube-proxy systemd service unit:

```bash
cat <<EOF | sudo tee /etc/systemd/system/kube-proxy.service
[Unit]
Description=Kubernetes Kube Proxy
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-proxy \\
  --config=/var/lib/kube-proxy/kube-proxy-config.yaml
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### 8. Start the Worker Services

Reload systemd and enable/start containerd, kubelet, and kube-proxy:

```bash
sudo systemctl daemon-reload
sudo systemctl enable containerd kubelet kube-proxy
sudo systemctl start containerd kubelet kube-proxy
```

> **Note:** Run these commands on each worker node: **worker-214**, **worker-241**, **worker-242**, **worker-243**, and **worker-244**.

## Verification

From one of your controller nodes (for example, controller-211), check that all worker nodes have registered:

```bash
ssh root@controller-211 kubectl get nodes --kubeconfig admin.kubeconfig
```

Expected output:

```bash
NAME         STATUS   ROLES    AGE   VERSION
worker-214   Ready    <none>   ...   v1.29.1
worker-241   Ready    <none>   ...   v1.29.1
worker-242   Ready    <none>   ...   v1.29.1
worker-243   Ready    <none>   ...   v1.29.1
worker-244   Ready    <none>   ...   v1.29.1
```

> **Additional Note:**  
> To ensure iptables works correctly with bridge-only traffic, load the `br_netfilter` module on all nodes (controllers and workers):

```bash
sudo modprobe br_netfilter
echo "br-netfilter" | sudo tee -a /etc/modules-load.d/modules.conf
sudo sysctl -w net.bridge.bridge-nf-call-iptables=1
```

Next: [Configuring kubectl for Remote Access](10-configuring-kubectl.md)

---

This revised guide has been updated for your Pittsburgh, PA environment with your actual worker node names and internal IP addressing as reflected in your netplan and hosts files.
