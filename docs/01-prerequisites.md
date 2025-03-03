Below is an updated version of the guide adapted to your topology. In this setup:

- Your upstream (ISP or router) default gateway is **10.10.12.1**.
- Your gateway VM (which acts as NAT, reverse proxy, and client tools host) has:
  - A **public interface** with IP **10.10.12.245/24** (used to reach the internet via 10.10.12.1).
  - A **private interface** with IP **192.168.1.245/24**.
- Your Kubernetes nodes (controllers and workers) use addresses in the **192.168.1.0/24** subnet.

The guide below reflects these changes.

---


# Prerequisites

## Proxmox Hypervisor

This tutorial is intended to be performed with a [Proxmox](https://proxmox.com/en/) hypervisor, but you can also use it with ESXi, KVM, Virtualbox, or another hypervisor.

> The compute resources required for this tutorial are 25GB of RAM and 140GB HDD (or SSD).

List of the VMs used in this tutorial:

```markdown

| Name          | Role                                   | vCPU | RAM  | Storage (thin) | IP                                               | OS     |
|---------------|----------------------------------------|------|------|----------------|--------------------------------------------------|--------|
| controller-211 | Controller                           | 2    | 4GB  | 20GB           | 192.168.1.211/24                                 | Ubuntu |
| controller-212 | Controller                           | 2    | 4GB  | 20GB           | 192.168.1.212/24                                 | Ubuntu |
| controller-213 | Controller                           | 2    | 4GB  | 20GB           | 192.168.1.213/24                                 | Ubuntu |
| worker-214    | Worker                                | 2    | 4GB  | 20GB           | 192.168.1.214/24                                 | Ubuntu |
| worker-241    | Worker                                | 2    | 4GB  | 20GB           | 192.168.1.241/24                                 | Ubuntu |
| worker-242    | Worker                                | 2    | 4GB  | 20GB           | 192.168.1.242/24                                 | Ubuntu |
| worker-243    | Worker                                | 2    | 4GB  | 20GB           | 192.168.1.243/24                                 | Ubuntu |
| worker-244    | Worker                                | 2    | 4GB  | 20GB           | 192.168.1.244/24                                 | Ubuntu |
| gateway-245   | Reverse Proxy, Client Tools, Gateway    | 1    | 1GB  | 20GB           | Private: 192.168.1.245/24 <br> Public: 10.10.12.245/24 | Debian |

```

On the Proxmox hypervisor, you might choose to add a `k8s-` prefix to the VM names if desired.

![proxmox vm list](images/proxmox-vm-list.PNG)

## Prepare the Environment

### Hypervisor Network

For this tutorial, you need 2 networks on your Proxmox hypervisor:

- A public network bridge (e.g., `vmbr0`).
- A private Kubernetes network bridge (e.g., `vmbr2`).

> Note: The pod networks will be defined later.

All Kubernetes nodes (workers and controllers) require only one network interface linked to the private Kubernetes network (`vmbr2`).

![proxmox vm hardware](images/proxmox-vm-hardware.PNG)

The reverse proxy / client tools / gateway VM needs 2 network interfaces:
- One linked to the private Kubernetes network (`vmbr2`).
- One linked to the public network (`vmbr0`).

![proxmox vm hardware](images/proxmox-vm-hardware-gw.PNG)

### Network Architecture

This diagram represents the network design:

- **Public Network:**
  - **Gateway VM public interface:** 10.10.12.245/24  
  - **Default gateway (upstream router):** 10.10.12.1
- **Private Kubernetes Network:**
  - **Kubernetes nodes (controllers and workers):** 192.168.1.0/24  
  - **Gateway VM private interface:** 192.168.1.245/24

![architecture network](images/architecture-network.png)

> If you want, you can define the IPv6 stack configuration.

---

### Gateway VM Installation

> The basic VM installation process is not the focus of this tutorial.
>
> For brevity, IPv6 is not configured here (but you may configure it if desired).

This VM is used as a NAT gateway for the private Kubernetes network, as a reverse proxy, and as a host for client tools (such as for certificate generation).

Perform the following steps on the gateway VM:

1. **Install the Debian netinst image**

   Download and install the latest [amd64 Debian netinst image](https://www.debian.org/CD/netinst/).

2. **Configure the network interfaces**

   Edit /etc/netplan/01-static.yaml (assuming your public interface is ens18 and your private interface is ens19). Replace the placeholders with your actual values:


```bash
   
root@debian245:~/.ssh# cat /etc/netplan/01-static.yaml

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

   > If you wish, you can also configure IPv6 and/or a different DNS resolver.

3. **Set the VM hostname**

   ```bash
   sudo hostnamectl set-hostname gateway-245
   ```

4. **Update the system**

   ```bash
   sudo apt-get update && sudo apt-get upgrade -y
   ```

5. **Install required packages**

   ```bash
   sudo apt-get install ssh vim tmux curl ntp iptables-persistent -y
   ```

6. **Enable and start SSH and NTP services**

   ```bash
   sudo systemctl enable ntp
   sudo systemctl start ntp
   sudo systemctl enable ssh
   sudo systemctl start ssh
   ```

7. **Enable IP routing**

   ```bash
   sudo sh -c "echo 'net.ipv4.ip_forward=1' >> /etc/sysctl.conf"
   sudo sh -c "echo '1' > /proc/sys/net/ipv4/ip_forward"
   ```

8. **Configure the iptables firewall**

   Create or edit `/etc/iptables/rules.v4` (assuming `ens18` is public and `ens19` is private):

   ```bash
   # Generated by xtables-save on [date]
   *nat
   -A POSTROUTING -o ens18 -j MASQUERADE
   COMMIT

   *filter
   -A INPUT -i lo -j ACCEPT
   # Allow SSH so that you don't lock yourself out
   -A INPUT -i ens18 -p tcp -m tcp --dport 22 -j ACCEPT
   -A INPUT -i ens18 -p tcp -m tcp --dport 80 -j ACCEPT
   -A INPUT -i ens18 -p tcp -m tcp --dport 443 -j ACCEPT
   -A INPUT -i ens18 -p tcp -m tcp --dport 6443 -j ACCEPT
   -A INPUT -i ens18 -p icmp -j ACCEPT
   # Allow established connections
   -A INPUT -m state --state RELATED,ESTABLISHED -j ACCEPT
   # Drop everything else on ens18
   -A INPUT -i ens18 -j DROP
   COMMIT
   ```

   > Again, you can configure IPv6 if needed.

9. **Activate the iptables rules**

   ```bash
   sudo iptables-restore < /etc/iptables/rules.v4
   ```

10. **Configure the `/etc/hosts` file**

    Edit `/etc/hosts` so it looks like this:

    ```plaintext
    127.0.0.1       localhost
    10.10.12.1    gateway-245.external gateway-245

    # IPv6 (optional)
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

11. **Reboot to confirm the network configuration**

    ```bash
    sudo reboot
    ```

---

### Kubernetes Nodes VM Installation

> The basic VM installation process is not the focus of this tutorial.
>
> IPv6 is not configured by default, but you can set it up if desired.

These VMs serve as Kubernetes nodes (controllers or workers). You can configure one, clone it, and then adjust the IP address and hostname for each clone.

For each node, perform the following steps:

1. **Install Ubuntu 22.04.3 LTS Server**

   Download and install the [Ubuntu 22.04.3 LTS Server image](https://releases.ubuntu.com/22.04/).

2. **Configure the network interface**

   Edit `/etc/netplan/00-installer-config.yaml` (assuming `ens19` is the private interface). For example:

   ```yaml
   # This is the network config written by 'subiquity'
   network:
     ethernets:
       ens19:
         addresses:
         - 192.168.1.X/24
         gateway4: 192.168.1.245
         nameservers:
           addresses:
           - 8.8.8.8
     version: 2
   ```

   Replace `192.168.1.X` with the appropriate IP for the node (e.g., `192.168.1.211` for controller-211).

3. **Set the hostname**

   For example, on controller-211:

   ```bash
   sudo hostnamectl set-hostname controller-211
   ```

4. **Update and upgrade the system**

   ```bash
   sudo apt-get update && sudo apt-get upgrade -y
   ```

5. **Install SSH and NTP**

   ```bash
   sudo apt-get install ssh ntp -y
   ```

6. **Enable and start SSH and NTP services**

   ```bash
   sudo systemctl enable ntp
   sudo systemctl start ntp
   sudo systemctl enable ssh
   sudo systemctl start ssh
   ```

7. **Configure the `/etc/hosts` file**

   An example for controller-211:

   ```plaintext
   127.0.0.1 localhost
   127.0.1.1 controller-211

   # IPv6 (optional)
   ::1             ip6-localhost ip6-loopback
   fe00::0         ip6-localnet
   ff00::0         ip6-mcastprefix
   ff02::1         ip6-allnodes
   ff02::2         ip6-allrouters

   10.10.12.1    gateway-245.external
   192.168.1.245   gateway-245

   192.168.1.212   controller-212
   192.168.1.213   controller-213

   192.168.1.214   worker-214
   192.168.1.241   worker-241
   192.168.1.242   worker-242
   192.168.1.243   worker-243
   192.168.1.244   worker-244
   ```

8. **Reboot to confirm the network configuration**

   ```bash
   sudo reboot
   ```

---

## Running Commands in Parallel with tmux

[tmux](https://github.com/tmux/tmux/wiki) can be used to run commands on multiple compute instances simultaneously. Many labs in this tutorial may require running the same commands across several nodes. To speed up the provisioning process, consider splitting a tmux window into multiple panes and enabling pane synchronization.

> To enable synchronize-panes, press `ctrl+b` followed by `shift+:`, then type `set synchronize-panes on` at the prompt. To disable it, type `set synchronize-panes off`.

![tmux screenshot](images/tmux-screenshot.png)

Next: [Installing the Client Tools](02-client-tools.md)
```

---

This adapted guide should align with your topology:
- **Public side:** Gateway VM uses 10.10.12.245 with upstream gateway 10.10.12.1.
- **Private side:** All Kubernetes nodes and the gateway’s private interface operate on 192.168.1.0/24, with the gateway at 192.168.1.245.

Feel free to adjust any details further as needed.
