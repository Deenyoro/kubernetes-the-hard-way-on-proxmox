Below is the revised guide for deploying the DNS Cluster Add-on in your Pittsburgh, PA environment. In this setup, your control plane and worker nodes are configured with the internal network (192.168.1.x), and your gateway VM (gateway-245) has the public IP 10.10.12.245. The instructions below assume you’ve already set up your cluster according to your netplan and hosts files.

---

# Deploying the DNS Cluster Add-on

In this lab you will deploy the DNS add-on, which provides DNS-based service discovery for applications running inside your Kubernetes cluster. This add-on is backed by CoreDNS.

## The DNS Cluster Add-on

Apply the CoreDNS deployment manifest:

```bash
kubectl apply -f https://raw.githubusercontent.com/DushanthaS/kubernetes-the-hard-way-on-proxmox/master/deployments/coredns.yaml
```

You should see output similar to:

```bash
serviceaccount/coredns created
clusterrole.rbac.authorization.k8s.io/system:coredns created
clusterrolebinding.rbac.authorization.k8s.io/system:coredns created
configmap/coredns created
deployment.extensions/coredns created
service/kube-dns created
```

Next, list the pods created by the CoreDNS (kube-dns) deployment:

```bash
kubectl get pods -l k8s-app=kube-dns -n kube-system
```

Wait a few seconds; you should eventually see something like:

```bash
NAME                       READY   STATUS    RESTARTS   AGE
coredns-699f8ddd77-94qv9   1/1     Running   0          20s
coredns-699f8ddd77-gtcgb   1/1     Running   0          20s
```

## Verification

Create a test deployment using BusyBox:

```bash
kubectl run busybox --image=busybox:1.28 --command -- sleep 3600
```

Check that the BusyBox pod is running:

```bash
kubectl get pods -l run=busybox
```

You should see output similar to:

```bash
NAME      READY   STATUS    RESTARTS   AGE
busybox   1/1     Running   0          3s
```

Retrieve the full name of the BusyBox pod:

```bash
POD_NAME=$(kubectl get pods -l run=busybox -o jsonpath="{.items[0].metadata.name}")
```

Now, execute a DNS lookup for the `kubernetes` service from inside the BusyBox pod:

```bash
kubectl exec -ti $POD_NAME -- nslookup kubernetes
```

The expected output should resemble:

```bash
Server:    10.32.0.10
Address 1: 10.32.0.10 kube-dns.kube-system.svc.cluster.local

Name:      kubernetes
Address 1: 10.32.0.1 kubernetes.default.svc.cluster.local
```

This confirms that CoreDNS is correctly resolving internal service names.

Let's break down the command:

```bash
kubectl exec -ti $POD_NAME -- nslookup kubernetes
```

- **`kubectl`**  
  The Kubernetes command-line tool that allows you to run commands against Kubernetes clusters.

- **`exec`**  
  This subcommand is used to execute a command in a container running in a pod.

- **`-t`**  
  Allocates a TTY (a pseudo-terminal) so that the command can run interactively. This is useful when you need formatted output or interactive behavior.

- **`-i`**  
  Keeps STDIN open even if not attached. This is useful when you need to interact with the process or when the command might require input.

- **`$POD_NAME`**  
  An environment variable that holds the name of the pod in which you want to run the command. This ensures you target the correct pod.

- **`--`**  
  Indicates the end of the command options for `kubectl exec`. Anything that follows is treated as the command to execute inside the container.

- **`nslookup`**  
  The command that will be executed inside the container. `nslookup` is a network utility used for querying the Domain Name System (DNS) to obtain domain name or IP address mapping information.

- **`kubernetes`**  
  The argument passed to `nslookup`. In this context, it tells `nslookup` to look up the DNS record for the hostname `kubernetes` (which, in Kubernetes clusters, is typically associated with the Kubernetes API server service).

This command, when run, will execute `nslookup kubernetes` inside the specified pod, allowing you to verify DNS resolution for the `kubernetes` service within the cluster.

Next: [Smoke Test](13-smoke-test.md)

---

This guide reflects your environment:
- **Gateway VM (gateway-245)** has public IP **10.10.12.245**.
- Internal hostnames and IPs are as defined in your `/etc/hosts` file.
- The service cluster IP (e.g., 10.32.0.1 for the Kubernetes API Server) and CoreDNS service IP (10.32.0.10) remain as per standard configuration.

You can now proceed with testing your DNS add-on and verifying that pod-to-service DNS resolution works throughout your cluster.
