Below is the revised guide for provisioning pod network routes in your Pittsburgh, PA environment. In your setup, each worker node is assigned a unique Pod CIDR and an internal IP on the 192.168.1.0/24 network. For example, you might assign:

- **worker-214**: Internal IP 192.168.1.214, Pod CIDR 10.200.1.0/24  
- **worker-241**: Internal IP 192.168.1.241, Pod CIDR 10.200.2.0/24  
- **worker-242**: Internal IP 192.168.1.242, Pod CIDR 10.200.3.0/24  
- **worker-243**: Internal IP 192.168.1.243, Pod CIDR 10.200.4.0/24  
- **worker-244**: Internal IP 192.168.1.244, Pod CIDR 10.200.5.0/24

Because pods on a node receive an IP from that node’s Pod CIDR, additional routes are needed so pods on one node can communicate with pods on another. **On each worker node, add routes for the Pod CIDRs of all other nodes—but not for its own Pod CIDR.**

For example, on **worker-214** (which has Pod CIDR 10.200.1.0/24), add the following routes:

```bash
ip route add 10.200.2.0/24 via 192.168.1.241  # Route to worker-241
ip route add 10.200.3.0/24 via 192.168.1.242  # Route to worker-242
ip route add 10.200.4.0/24 via 192.168.1.243  # Route to worker-243
ip route add 10.200.5.0/24 via 192.168.1.244  # Route to worker-244
```

> **Note:** If you see messages such as `RTNETLINK answers: File exists`, it means the route already exists and can be safely ignored.

### Verification

On any worker node, list the routes with:

```bash
ip route
```

For example, on worker-214 you might see output similar to:

```bash
default via 192.168.1.245 dev ens18 proto static
10.200.1.0/24 via 192.168.1.241 dev ens18
10.200.2.0/24 via 192.168.1.242 dev ens18
10.200.3.0/24 via 192.168.1.243 dev ens18
10.200.4.0/24 via 192.168.1.244 dev ens18
192.168.1.0/24 dev ens18 proto kernel scope link src 192.168.1.214
```

### Making Routes Persistent

To ensure these routes persist after a reboot, update your network configuration on each worker node. For example, on **worker-214** (which should not include its own Pod CIDR route), you could edit the netplan configuration (e.g., `/etc/netplan/00-installer-config.yaml`) as follows:

```yaml
# Example netplan config for worker-214
network:
  version: 2
  renderer: networkd
  ethernets:
    ens18:
      addresses:
        - 192.168.1.214/24
      gateway4: 192.168.1.245
      nameservers:
        addresses:
          - 8.8.8.8
          - 8.8.4.4
      routes:
        - to: 10.200.2.0/24
          via: 192.168.1.241
        - to: 10.200.3.0/24
          via: 192.168.1.242
        - to: 10.200.4.0/24
          via: 192.168.1.243
        - to: 10.200.5.0/24
          via: 192.168.1.244
```

Adjust the routes for each worker node accordingly—omitting the route for the Pod CIDR that belongs to the node itself.

Below are sample netplan YAML configuration files for each worker node (excluding worker‑214 which was provided earlier). These files assume that:

- The worker nodes are on the internal network 192.168.1.0/24.
- Your gateway (for the default route) is at 192.168.1.245.
- You’ve assigned the following Pod CIDRs:
  - **worker‑214:** 10.200.1.0/24  
  - **worker‑241:** 10.200.2.0/24  
  - **worker‑242:** 10.200.3.0/24  
  - **worker‑243:** 10.200.4.0/24  
  - **worker‑244:** 10.200.5.0/24

For each worker node, you must add routes for all the other workers’ Pod CIDRs (but omit the route for its own Pod CIDR).

---

### Worker-241 (Internal IP: 192.168.1.241, Pod CIDR: 10.200.2.0/24)

```yaml
# /etc/netplan/00-installer-config.yaml for worker-241
network:
  version: 2
  renderer: networkd
  ethernets:
    ens18:
      addresses:
        - 192.168.1.241/24
      gateway4: 192.168.1.245
      nameservers:
        addresses:
          - 8.8.8.8
          - 8.8.4.4
      routes:
        - to: 10.200.1.0/24   # worker-214
          via: 192.168.1.214
        - to: 10.200.3.0/24   # worker-242
          via: 192.168.1.242
        - to: 10.200.4.0/24   # worker-243
          via: 192.168.1.243
        - to: 10.200.5.0/24   # worker-244
          via: 192.168.1.244
```

---

### Worker-242 (Internal IP: 192.168.1.242, Pod CIDR: 10.200.3.0/24)

```yaml
# /etc/netplan/00-installer-config.yaml for worker-242
network:
  version: 2
  renderer: networkd
  ethernets:
    ens18:
      addresses:
        - 192.168.1.242/24
      gateway4: 192.168.1.245
      nameservers:
        addresses:
          - 8.8.8.8
          - 8.8.4.4
      routes:
        - to: 10.200.1.0/24   # worker-214
          via: 192.168.1.214
        - to: 10.200.2.0/24   # worker-241
          via: 192.168.1.241
        - to: 10.200.4.0/24   # worker-243
          via: 192.168.1.243
        - to: 10.200.5.0/24   # worker-244
          via: 192.168.1.244
```

---

### Worker-243 (Internal IP: 192.168.1.243, Pod CIDR: 10.200.4.0/24)

```yaml
# /etc/netplan/00-installer-config.yaml for worker-243
network:
  version: 2
  renderer: networkd
  ethernets:
    ens18:
      addresses:
        - 192.168.1.243/24
      gateway4: 192.168.1.245
      nameservers:
        addresses:
          - 8.8.8.8
          - 8.8.4.4
      routes:
        - to: 10.200.1.0/24   # worker-214
          via: 192.168.1.214
        - to: 10.200.2.0/24   # worker-241
          via: 192.168.1.241
        - to: 10.200.3.0/24   # worker-242
          via: 192.168.1.242
        - to: 10.200.5.0/24   # worker-244
          via: 192.168.1.244
```

---

### Worker-244 (Internal IP: 192.168.1.244, Pod CIDR: 10.200.5.0/24)

```yaml
# /etc/netplan/00-installer-config.yaml for worker-244
network:
  version: 2
  renderer: networkd
  ethernets:
    ens18:
      addresses:
        - 192.168.1.244/24
      gateway4: 192.168.1.245
      nameservers:
        addresses:
          - 8.8.8.8
          - 8.8.4.4
      routes:
        - to: 10.200.1.0/24   # worker-214
          via: 192.168.1.214
        - to: 10.200.2.0/24   # worker-241
          via: 192.168.1.241
        - to: 10.200.3.0/24   # worker-242
          via: 192.168.1.242
        - to: 10.200.4.0/24   # worker-243
          via: 192.168.1.243
```

---

**Usage:**  
Place the appropriate YAML file on each worker node (e.g., `/etc/netplan/00-installer-config.yaml`), then apply the configuration with:

```bash
sudo netplan apply
```

These configurations ensure that on each worker node, routes exist to reach the Pod CIDRs of the other worker nodes, enabling inter-node pod communication.

Next: [Deploying the DNS Cluster Add-on](12-dns-addon.md)
