Below is the revised guide for bootstrapping the Kubernetes Control Plane in your environment. In this setup:

- **Controller Nodes:**  
  - **controller-211** with internal IP: **192.168.1.211**  
  - **controller-212** with internal IP: **192.168.1.212**  
  - **controller-213** with internal IP: **192.168.1.213**

- **Gateway VM:**  
  - Public IP: **10.10.12.245** (this will be used as the API Server’s external endpoint)  
  - Private IP: **192.168.1.245**

The following commands should be run on each controller node (i.e. controller-211, controller-212, and controller-213).

---

# Bootstrapping the Kubernetes Control Plane

In this lab you will bootstrap the Kubernetes control plane across three VM instances and configure it for high availability. You will also create a load balancer that exposes the Kubernetes API Servers to remote clients. The following components will be installed on each controller node: Kubernetes API Server, Scheduler, and Controller Manager.

## Prerequisites

The commands in this lab must be run on each controller instance: **controller-211**, **controller-212**, and **controller-213**. Log in to each using `ssh`. For example:

```bash
ssh root@controller-211
```

### Running Commands in Parallel with tmux

[tmux](https://github.com/tmux/tmux/wiki) can be used to run commands on multiple nodes simultaneously. See the [Running commands in parallel with tmux](01-prerequisites.md#running-commands-in-parallel-with-tmux) section for more details.

## Provision the Kubernetes Control Plane

Create the Kubernetes configuration directory:

```bash
sudo mkdir -p /etc/kubernetes/config
```

### Download and Install the Kubernetes Controller Binaries

Download the official Kubernetes release binaries:

```bash
wget -q --show-progress --https-only --timestamping \
  "https://storage.googleapis.com/kubernetes-release/release/v1.29.1/bin/linux/amd64/kube-apiserver" \
  "https://storage.googleapis.com/kubernetes-release/release/v1.29.1/bin/linux/amd64/kube-controller-manager" \
  "https://storage.googleapis.com/kubernetes-release/release/v1.29.1/bin/linux/amd64/kube-scheduler" \
  "https://storage.googleapis.com/kubernetes-release/release/v1.29.1/bin/linux/amd64/kubectl"
```

Make the binaries executable and install them:

```bash
chmod +x kube-apiserver kube-controller-manager kube-scheduler kubectl
sudo mv kube-apiserver kube-controller-manager kube-scheduler kubectl /usr/local/bin/
```

### Configure the Kubernetes API Server

Create the directory for Kubernetes configuration files and move the required certificates:

```bash
sudo mkdir -p /var/lib/kubernetes/

sudo mv ca.pem ca-key.pem kubernetes-key.pem kubernetes.pem \
  service-account-key.pem service-account.pem \
  encryption-config.yaml /var/lib/kubernetes/
```

Each controller node will use its internal IP address to advertise the API Server. On each controller, define the INTERNAL_IP variable to match its IP. For example, on **controller-211** run:

```bash
INTERNAL_IP=192.168.1.211
```

> On controller-212 and controller-213, set INTERNAL_IP to 192.168.1.212 and 192.168.1.213 respectively.

Define the static public IP address (provided by your load balancer on gateway-245):

```bash
KUBERNETES_PUBLIC_ADDRESS=10.10.12.245
```

Create the `kube-apiserver.service` systemd unit file:

```bash
cat <<EOF | sudo tee /etc/systemd/system/kube-apiserver.service
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-apiserver \\
  --advertise-address=\${INTERNAL_IP} \\
  --allow-privileged=true \\
  --apiserver-count=3 \\
  --audit-log-maxage=30 \\
  --audit-log-maxbackup=3 \\
  --audit-log-maxsize=100 \\
  --audit-log-path=/var/log/audit.log \\
  --authorization-mode=Node,RBAC \\
  --bind-address=0.0.0.0 \\
  --client-ca-file=/var/lib/kubernetes/ca.pem \\
  --enable-admission-plugins=NamespaceLifecycle,NodeRestriction,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota \\
  --etcd-cafile=/var/lib/kubernetes/ca.pem \\
  --etcd-certfile=/var/lib/kubernetes/kubernetes.pem \\
  --etcd-keyfile=/var/lib/kubernetes/kubernetes-key.pem \\
  --etcd-servers=https://192.168.1.211:2379,https://192.168.1.212:2379,https://192.168.1.213:2379 \\
  --event-ttl=1h \\
  --encryption-provider-config=/var/lib/kubernetes/encryption-config.yaml \\
  --kubelet-certificate-authority=/var/lib/kubernetes/ca.pem \\
  --kubelet-client-certificate=/var/lib/kubernetes/kubernetes.pem \\
  --kubelet-client-key=/var/lib/kubernetes/kubernetes-key.pem \\
  --runtime-config=api/all=true \\
  --service-account-key-file=/var/lib/kubernetes/service-account.pem \\
  --service-account-signing-key-file=/var/lib/kubernetes/service-account-key.pem \\
  --service-account-issuer=https://\${KUBERNETES_PUBLIC_ADDRESS}:6443 \\
  --service-cluster-ip-range=10.32.0.0/24 \\
  --service-node-port-range=30000-32767 \\
  --tls-cert-file=/var/lib/kubernetes/kubernetes.pem \\
  --tls-private-key-file=/var/lib/kubernetes/kubernetes-key.pem \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### Configure the Kubernetes Controller Manager

Move the `kube-controller-manager` kubeconfig into place:

```bash
sudo mv kube-controller-manager.kubeconfig /var/lib/kubernetes/
```

Create the `kube-controller-manager.service` systemd unit file:

```bash
cat <<EOF | sudo tee /etc/systemd/system/kube-controller-manager.service
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-controller-manager \\
  --bind-address=0.0.0.0 \\
  --cluster-cidr=10.200.0.0/16 \\
  --cluster-name=kubernetes \\
  --cluster-signing-cert-file=/var/lib/kubernetes/ca.pem \\
  --cluster-signing-key-file=/var/lib/kubernetes/ca-key.pem \\
  --kubeconfig=/var/lib/kubernetes/kube-controller-manager.kubeconfig \\
  --leader-elect=true \\
  --root-ca-file=/var/lib/kubernetes/ca.pem \\
  --service-account-private-key-file=/var/lib/kubernetes/service-account-key.pem \\
  --service-cluster-ip-range=10.32.0.0/24 \\
  --use-service-account-credentials=true \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### Configure the Kubernetes Scheduler

Move the `kube-scheduler` kubeconfig into place:

```bash
sudo mv kube-scheduler.kubeconfig /var/lib/kubernetes/
```

Create the scheduler configuration file:

```bash
cat <<EOF | sudo tee /etc/kubernetes/config/kube-scheduler.yaml
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
clientConnection:
  kubeconfig: "/var/lib/kubernetes/kube-scheduler.kubeconfig"
leaderElection:
  leaderElect: true
EOF
```

Create the `kube-scheduler.service` systemd unit file:

```bash
cat <<EOF | sudo tee /etc/systemd/system/kube-scheduler.service
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-scheduler \\
  --config=/etc/kubernetes/config/kube-scheduler.yaml \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### Start the Controller Services

Reload systemd, enable, and start the control plane services:

```bash
sudo systemctl daemon-reload
sudo systemctl enable kube-apiserver kube-controller-manager kube-scheduler
sudo systemctl start kube-apiserver kube-controller-manager kube-scheduler
```

> Allow up to 10 seconds for the Kubernetes API Server to fully initialize.

### Verification

Verify the control plane is up by running:

```bash
kubectl cluster-info --kubeconfig admin.kubeconfig
```

You should see output similar to:

```bash
Kubernetes control plane is running at https://127.0.0.1:6443
```

Test the API Server health check:

```bash
curl -kH "Host: kubernetes.default.svc.cluster.local" -i https://127.0.0.1:6443/healthz
```

Expected output:

```bash
HTTP/2 200
content-type: text/plain; charset=utf-8
x-content-type-options: nosniff
content-length: 2

ok
```

## RBAC for Kubelet Authorization

(Instructions for configuring RBAC for kubelet authorization follow here, which affect the entire cluster and are run once from one controller.)

## The Kubernetes Frontend Load Balancer

(Instructions to provision an Nginx load balancer to front the API Servers on the gateway VM follow here.)

Next: [Bootstrapping the Kubernetes Worker Nodes](09-bootstrapping-kubernetes-workers.md)
```

---

### Additional Reference: Your Network Configuration on gateway-245

For reference, here’s your netplan and hosts configuration on **gateway-245**:

```bash
# /etc/netplan/01-static.yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    ens18:
      addresses:
        - 10.10.12.245/24
      routes:
        - to: default
          via: 10.10.12.1
      nameservers:
        addresses:
          - 8.8.8.8
          - 8.8.4.4
    ens19:
      addresses:
        - 192.168.1.245/24
```

```plaintext
# /etc/hosts on gateway-245
127.0.0.1       localhost
10.10.12.245    gateway-245.external gateway-245

# IPv6 settings:
::1             localhost ip6-localhost ip6-loopback
ff02::1         ip6-allnodes
ff02::2         ip6-allrouters

192.168.1.211   controller-211
192.168.1.212   controller-212
192.168.1.213   controller-213

192.168.1.214   worker-214
192.168.1.241   worker-241
192.168.1.242   worker-242
192.168.1.243   worker-243
192.168.1.244   worker-244
```

This revised guide reflects your environment and ensures that your control plane components are configured with the correct internal IP addresses and load-balanced public endpoint.
