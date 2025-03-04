```markdown
# Provisioning Pod Network Routes in Pittsburgh, PA

In this guide, we will configure static routes on each Kubernetes worker node so that pods on different nodes can communicate. When each worker node has its own Pod CIDR, you must tell every other worker node how to reach that CIDR. Below is an example setup:

- **worker-214**:  
  - Node IP: `192.168.1.214`  
  - Pod CIDR: `10.200.1.0/24`
- **worker-241**:  
  - Node IP: `192.168.1.241`  
  - Pod CIDR: `10.200.2.0/24`
- **worker-242**:  
  - Node IP: `192.168.1.242`  
  - Pod CIDR: `10.200.3.0/24`
- **worker-243**:  
  - Node IP: `192.168.1.243`  
  - Pod CIDR: `10.200.4.0/24`
- **worker-244**:  
  - Node IP: `192.168.1.244`  
  - Pod CIDR: `10.200.5.0/24`

Because pods on a node receive IPs from that node’s Pod CIDR, routes are needed so pods on one node can reach pods on another. On each worker node, **add routes for the other nodes’ Pod CIDRs** but not for its own Pod CIDR.

---

## 1. Add Routes Manually (Temporary)

You can first test routes manually with commands like:

```bash
ip route add 10.200.2.0/24 via 192.168.1.241
ip route add 10.200.3.0/24 via 192.168.1.242
ip route add 10.200.4.0/24 via 192.168.1.243
ip route add 10.200.5.0/24 via 192.168.1.244
```

These routes will last until the node reboots or the network restarts. If you get a `RTNETLINK answers: File exists`, it typically means the route was already added and can be ignored.

You can verify your routes with:

```bash
ip route
```

---

## 2. Persisting Routes with Netplan

To make these routes permanent, you should update your netplan configuration on each node. This involves:

1. Removing (or not using) the deprecated `gateway4` option.
2. Specifying a default route as `to: 0.0.0.0/0`.
3. Setting correct routes for the other nodes’ Pod CIDRs.
4. Using the **`eth1`** interface (based on your environment).
5. Securing the netplan file permissions (`chmod 600`).

Below are **complete** sample files for each worker node. Save these as `/etc/netplan/00-installer-config.yaml` (or a similarly named `.yaml` file) **on the corresponding worker**, then run `sudo chmod 600 /etc/netplan/00-installer-config.yaml` to secure the file. Finally, apply the changes with `sudo netplan apply`.

---

### 2.1 Configuration for Worker-214

- **Node IP:** `192.168.1.214`  
- **Pod CIDR:** `10.200.1.0/24`  
- **Other Nodes’ Pod CIDRs:** `10.200.2.0/24`, `10.200.3.0/24`, `10.200.4.0/24`, `10.200.5.0/24`  

```yaml
# /etc/netplan/00-installer-config.yaml (worker-214)
network:
  version: 2
  renderer: networkd
  ethernets:
    eth1:
      addresses:
        - 192.168.1.214/24
      nameservers:
        addresses:
          - 8.8.8.8
          - 8.8.4.4
      routes:
        # Default route
        - to: 0.0.0.0/0
          via: 192.168.1.245
          metric: 100

        # Routes to other workers' Pod CIDRs (exclude own Pod CIDR: 10.200.1.0/24)
        - to: 10.200.2.0/24
          via: 192.168.1.241
        - to: 10.200.3.0/24
          via: 192.168.1.242
        - to: 10.200.4.0/24
          via: 192.168.1.243
        - to: 10.200.5.0/24
          via: 192.168.1.244
```

---

### 2.2 Configuration for Worker-241

- **Node IP:** `192.168.1.241`  
- **Pod CIDR:** `10.200.2.0/24`  
- **Other Nodes’ Pod CIDRs:** `10.200.1.0/24`, `10.200.3.0/24`, `10.200.4.0/24`, `10.200.5.0/24`  

```yaml
# /etc/netplan/00-installer-config.yaml (worker-241)
network:
  version: 2
  renderer: networkd
  ethernets:
    eth1:
      addresses:
        - 192.168.1.241/24
      nameservers:
        addresses:
          - 8.8.8.8
          - 8.8.4.4
      routes:
        # Default route
        - to: 0.0.0.0/0
          via: 192.168.1.245
          metric: 100

        # Routes to other workers' Pod CIDRs (exclude own Pod CIDR: 10.200.2.0/24)
        - to: 10.200.1.0/24
          via: 192.168.1.214
        - to: 10.200.3.0/24
          via: 192.168.1.242
        - to: 10.200.4.0/24
          via: 192.168.1.243
        - to: 10.200.5.0/24
          via: 192.168.1.244
```

---

### 2.3 Configuration for Worker-242

- **Node IP:** `192.168.1.242`  
- **Pod CIDR:** `10.200.3.0/24`  
- **Other Nodes’ Pod CIDRs:** `10.200.1.0/24`, `10.200.2.0/24`, `10.200.4.0/24`, `10.200.5.0/24`  

```yaml
# /etc/netplan/00-installer-config.yaml (worker-242)
network:
  version: 2
  renderer: networkd
  ethernets:
    eth1:
      addresses:
        - 192.168.1.242/24
      nameservers:
        addresses:
          - 8.8.8.8
          - 8.8.4.4
      routes:
        # Default route
        - to: 0.0.0.0/0
          via: 192.168.1.245
          metric: 100

        # Routes to other workers' Pod CIDRs (exclude own Pod CIDR: 10.200.3.0/24)
        - to: 10.200.1.0/24
          via: 192.168.1.214
        - to: 10.200.2.0/24
          via: 192.168.1.241
        - to: 10.200.4.0/24
          via: 192.168.1.243
        - to: 10.200.5.0/24
          via: 192.168.1.244
```

---

### 2.4 Configuration for Worker-243

- **Node IP:** `192.168.1.243`  
- **Pod CIDR:** `10.200.4.0/24`  
- **Other Nodes’ Pod CIDRs:** `10.200.1.0/24`, `10.200.2.0/24`, `10.200.3.0/24`, `10.200.5.0/24`  

```yaml
# /etc/netplan/00-installer-config.yaml (worker-243)
network:
  version: 2
  renderer: networkd
  ethernets:
    eth1:
      addresses:
        - 192.168.1.243/24
      nameservers:
        addresses:
          - 8.8.8.8
          - 8.8.4.4
      routes:
        # Default route
        - to: 0.0.0.0/0
          via: 192.168.1.245
          metric: 100

        # Routes to other workers' Pod CIDRs (exclude own Pod CIDR: 10.200.4.0/24)
        - to: 10.200.1.0/24
          via: 192.168.1.214
        - to: 10.200.2.0/24
          via: 192.168.1.241
        - to: 10.200.3.0/24
          via: 192.168.1.242
        - to: 10.200.5.0/24
          via: 192.168.1.244
```

---

### 2.5 Configuration for Worker-244

- **Node IP:** `192.168.1.244`  
- **Pod CIDR:** `10.200.5.0/24`  
- **Other Nodes’ Pod CIDRs:** `10.200.1.0/24`, `10.200.2.0/24`, `10.200.3.0/24`, `10.200.4.0/24`  

```yaml
# /etc/netplan/00-installer-config.yaml (worker-244)
network:
  version: 2
  renderer: networkd
  ethernets:
    eth1:
      addresses:
        - 192.168.1.244/24
      nameservers:
        addresses:
          - 8.8.8.8
          - 8.8.4.4
      routes:
        # Default route
        - to: 0.0.0.0/0
          via: 192.168.1.245
          metric: 100

        # Routes to other workers' Pod CIDRs (exclude own Pod CIDR: 10.200.5.0/24)
        - to: 10.200.1.0/24
          via: 192.168.1.214
        - to: 10.200.2.0/24
          via: 192.168.1.241
        - to: 10.200.3.0/24
          via: 192.168.1.242
        - to: 10.200.4.0/24
          via: 192.168.1.243
```

---

## 3. Apply the Configuration

1. Place each netplan YAML file onto its corresponding worker node at `/etc/netplan/00-installer-config.yaml`.
2. Secure the file:

   ```bash
   sudo chmod 600 /etc/netplan/00-installer-config.yaml
   ```
3. Apply the configuration:

   ```bash
   sudo netplan apply
   ```

Check the routes again (`ip route`) to ensure they are added.

---

## Next Steps

With routes in place, pods on different nodes should be able to communicate. Continue by [Deploying the DNS Cluster Add-on](12-dns-addon.md) to enable service name resolution within your cluster.

```
