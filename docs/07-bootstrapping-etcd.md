Below is the revised guide for bootstrapping the etcd cluster in your environment. In your setup, the controller nodes are named **controller-211**, **controller-212**, and **controller-213**, with the following internal IP addresses:

- **controller-211**: 192.168.1.211  
- **controller-212**: 192.168.1.212  
- **controller-213**: 192.168.1.213

Run these commands on each controller node (using, for example, `ssh root@controller-211`).

---


# Bootstrapping the etcd Cluster

Kubernetes components are stateless and store cluster state in [etcd](https://github.com/etcd-io/etcd). In this lab you will bootstrap a three-node etcd cluster and configure it for high availability and secure remote access.

## Prerequisites

The commands in this lab must be run on each controller instance: **controller-211**, **controller-212**, and **controller-213**. Log in to each controller instance using `ssh`. For example, for controller-211:

```bash
ssh root@controller-211
```

### Running commands in parallel with tmux

[tmux](https://github.com/tmux/tmux/wiki) can be used to run commands on multiple instances simultaneously. See the [Running commands in parallel with tmux](01-prerequisites.md#running-commands-in-parallel-with-tmux) section in the Prerequisites lab.

## Bootstrapping an etcd Cluster Member

### Download and Install the etcd Binaries

Download the official etcd release binaries from the [etcd GitHub project](https://github.com/etcd-io/etcd):

```bash
wget -q --show-progress --https-only --timestamping \
  "https://github.com/etcd-io/etcd/releases/download/v3.5.12/etcd-v3.5.12-linux-amd64.tar.gz"
```

Extract and install the `etcd` server and the `etcdctl` utility:

```bash
tar -xvf etcd-v3.5.12-linux-amd64.tar.gz
sudo mv etcd-v3.5.12-linux-amd64/etcd* /usr/local/bin/
```

### Configure the etcd Server

Create necessary directories and copy the required certificates (which you generated earlier):

```bash
sudo mkdir -p /etc/etcd /var/lib/etcd
sudo cp ca.pem kubernetes-key.pem kubernetes.pem /etc/etcd/
```

Each etcd member will use its internal IP address to serve client requests and communicate with cluster peers. On each controller node, define the INTERNAL_IP variable using the node’s IP address. For example, on **controller-211**:

```bash
INTERNAL_IP=192.168.1.211
```

> **Note:** On controller-212 and controller-213, set INTERNAL_IP to 192.168.1.212 and 192.168.1.213 respectively.

Each etcd member must have a unique name. Set the etcd name to match the hostname:

```bash
ETCD_NAME=$(hostname -s)
```

Create the systemd unit file for etcd:

```bash
cat <<EOF | sudo tee /etc/systemd/system/etcd.service
[Unit]
Description=etcd
Documentation=https://github.com/coreos

[Service]
Type=notify
ExecStart=/usr/local/bin/etcd \\
  --name ${ETCD_NAME} \\
  --cert-file=/etc/etcd/kubernetes.pem \\
  --key-file=/etc/etcd/kubernetes-key.pem \\
  --peer-cert-file=/etc/etcd/kubernetes.pem \\
  --peer-key-file=/etc/etcd/kubernetes-key.pem \\
  --trusted-ca-file=/etc/etcd/ca.pem \\
  --peer-trusted-ca-file=/etc/etcd/ca.pem \\
  --peer-client-cert-auth \\
  --client-cert-auth \\
  --initial-advertise-peer-urls https://${INTERNAL_IP}:2380 \\
  --listen-peer-urls https://${INTERNAL_IP}:2380 \\
  --listen-client-urls https://${INTERNAL_IP}:2379,https://127.0.0.1:2379 \\
  --advertise-client-urls https://${INTERNAL_IP}:2379 \\
  --initial-cluster-token etcd-cluster-0 \\
  --initial-cluster controller-211=https://192.168.1.211:2380,controller-212=https://192.168.1.212:2380,controller-213=https://192.168.1.213:2380 \\
  --initial-cluster-state new \\
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### Start the etcd Server

Reload the systemd configuration, enable, and start etcd:

```bash
sudo systemctl daemon-reload
sudo systemctl enable etcd
sudo systemctl start etcd
```

> **Reminder:** Repeat these steps on **controller-211**, **controller-212**, and **controller-213**.

## Verification

List the etcd cluster members by running the following command on any controller:

```bash
sudo ETCDCTL_API=3 etcdctl member list \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/etcd/ca.pem \
  --cert=/etc/etcd/kubernetes.pem \
  --key=/etc/etcd/kubernetes-key.pem
```

A successful output should list your three members similar to:

```bash
<member-id>, started, controller-211, https://192.168.1.211:2380, https://192.168.1.211:2379
<member-id>, started, controller-212, https://192.168.1.212:2380, https://192.168.1.212:2379
<member-id>, started, controller-213, https://192.168.1.213:2380, https://192.168.1.213:2379
```

Next: [Bootstrapping the Kubernetes Control Plane](08-bootstrapping-kubernetes-controllers.md)
