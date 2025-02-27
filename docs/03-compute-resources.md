---

# Provisioning Compute Resources

Kubernetes requires a set of machines to host both the control plane and the worker nodes. In this lab you’ll review (and adjust if necessary) the configurations you defined in the prerequisites.

## Networking

The Kubernetes [networking model](https://kubernetes.io/docs/concepts/cluster-administration/networking/#kubernetes-model) assumes a flat network in which all containers and nodes can communicate with each other. (Network policies can later restrict this communication if needed.)

### Virtual Private Cloud Network

In the prerequisites, we set up a private network—**192.168.1.0/24**—to host our Kubernetes nodes. This “VPC-like” network supports up to 253 nodes (with one IP reserved for the gateway).

### Pods Network Ranges

Containers (Pods) on each worker need their own subnetworks. We will use the **10.200.0.0/16** private range to create Pod subnets. For example, if you have five workers you might allocate:

- worker-214: 10.200.0.0/24  
- worker-241: 10.200.1.0/24  
- worker-242: 10.200.2.0/24  
- worker-243: 10.200.3.0/24  
- worker-244: 10.200.4.0/24  

Adjust the assignments according to your actual node count.

### Firewall Rules

Within the Kubernetes private network (linked via your Proxmox bridge, e.g. `vmbr8`), all traffic is permitted. On the **gateway-245** VM, the firewall is configured to NAT traffic and allow the following inbound protocols from external networks:  
- ICMP  
- TCP/22 (SSH)  
- TCP/80 (HTTP)  
- TCP/443 (HTTPS)  
- TCP/6443 (Kubernetes API)

To view the firewall rules on **gateway-245** (assuming `ens18` is the public interface):

```bash
sudo iptables -L INPUT -v -n
```

You should see rules similar to:

```bash
Chain INPUT (policy ACCEPT)
 pkts bytes target     prot opt in     out     source               destination
  ...  ...  ACCEPT     all  --  lo     *       0.0.0.0/0            0.0.0.0/0
  ...  ...  ACCEPT     tcp  --  ens18  *       0.0.0.0/0            0.0.0.0/0            tcp dpt:22
  ...  ...  ACCEPT     tcp  --  ens18  *       0.0.0.0/0            0.0.0.0/0            tcp dpt:80
  ...  ...  ACCEPT     tcp  --  ens18  *       0.0.0.0/0            0.0.0.0/0            tcp dpt:443
  ...  ...  ACCEPT     tcp  --  ens18  *       0.0.0.0/0            0.0.0.0/0            tcp dpt:6443
  ...  ...  ACCEPT     icmp --  ens18  *       0.0.0.0/0            0.0.0.0/0
  ...  ...  ACCEPT     all  --  *      *       0.0.0.0/0            0.0.0.0/0            state RELATED,ESTABLISHED
  ...  ...  DROP       all  --  ens18  *       0.0.0.0/0            0.0.0.0/0
```

### Kubernetes Public IP Address

A public IP must be assigned to the public interface of the **gateway-245** VM. In our setup, this is **10.10.12.245/24**, with the upstream default gateway set as **10.10.12.1**.

### Verification

On each VM, check the active IP addresses:

```bash
ip a
```

> Example output (from, say, controller-211):
>
> ```bash
> 2: ens18: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 ...
>     inet 192.168.1.211/24 brd 192.168.1.255 scope global ens18
>     inet6 fe80::.../64 scope link
> ```

From the **gateway-245** VM, verify connectivity by pinging each node. For example:

```bash
ping -c1 controller-211
ping -c1 worker-214
```

(Repeat for each controller and worker as defined in your `/etc/hosts` file.)

## Configuring SSH Access

SSH is used to manage the controller and worker nodes.

1. **Generate an SSH Key on gateway-245**

   On the **gateway-245** VM, run:

   ```bash
   ssh-keygen
   ```

   Follow the prompts (for example, if you’re using the default file location and optionally a passphrase).

2. **Display and Copy the Public Key**

   Print your public key to the terminal:

   ```bash
   cat ~/.ssh/id_rsa.pub
   ```

   Copy the output.

3. **Deploy the Public Key on All Nodes**

   On each controller and worker node, create the SSH directory and paste the public key into `/root/.ssh/authorized_keys`:

   ```bash
   mkdir -p /root/.ssh
   vi /root/.ssh/authorized_keys
   ```

   Paste the copied key and save the file.

4. **Test SSH Connectivity**

   From **gateway-245**, try connecting to a node (for example, controller-211):

   ```bash
   ssh root@controller-211
   ```

   You should see a welcome message from the node. Exit the session by typing:

   ```bash
   exit
   ```

Next: [Provisioning a CA and Generating TLS Certificates](04-certificate-authority.md)


---

This guide now reflects your topology:  
- **gateway-245** has the public IP **10.10.12.245/24** (with an upstream gateway of **10.10.12.1**) and a private IP **192.168.1.245/24**.  
- All Kubernetes nodes use the **192.168.1.0/24** network.  
- The pod networks are planned in the **10.200.0.0/16** range.

Feel free to adjust any subnets or hostnames further to fit your exact environment.
