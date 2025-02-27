Below is the revised guide for generating Kubernetes configuration files for authentication, adapted to your environment. In your setup:

- Your gateway VM (**gateway-245**) has a public IP of **10.10.12.245** (acting as your external load balancer) and a private IP of **192.168.1.245**.
- Your controllers are **controller-211**, **controller-212**, and **controller-213**.
- Your worker nodes are **worker-214**, **worker-241**, **worker-242**, **worker-243**, and **worker-244**.
- In this guide, we assume that the Kubernetes API Server will be reached externally via **10.10.12.245:6443** (for workers) while on controllers local requests use **127.0.0.1:6443**.
- Region details are updated to US, Pittsburgh, PA.

---

# Generating Kubernetes Configuration Files for Authentication

In this lab you will generate [Kubernetes configuration files](https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/) (kubeconfigs) that enable Kubernetes clients to locate and authenticate to the Kubernetes API Server.

## Client Authentication Configs

You will generate kubeconfig files for the kubelet, kube-proxy, kube-controller-manager, kube-scheduler, and the admin user.

### Kubernetes Public IP Address

Each kubeconfig requires an API Server endpoint. In our environment we use the external public IP of the gateway VM. Define the public address:

```bash
KUBERNETES_PUBLIC_ADDRESS=10.10.12.245
```

### The kubelet Kubernetes Configuration File

For kubelet configurations, the client certificate must match the kubelet's node name. Generate a kubeconfig file for each worker node:

```bash
for instance in worker-214 worker-241 worker-242 worker-243 worker-244; do
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://${KUBERNETES_PUBLIC_ADDRESS}:6443 \
    --kubeconfig=${instance}.kubeconfig

  kubectl config set-credentials system:node:${instance} \
    --client-certificate=${instance}.pem \
    --client-key=${instance}-key.pem \
    --embed-certs=true \
    --kubeconfig=${instance}.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:node:${instance} \
    --kubeconfig=${instance}.kubeconfig

  kubectl config use-context default --kubeconfig=${instance}.kubeconfig
done
```

This produces the following kubeconfig files:
- `worker-214.kubeconfig`
- `worker-241.kubeconfig`
- `worker-242.kubeconfig`
- `worker-243.kubeconfig`
- `worker-244.kubeconfig`

### The kube-proxy Kubernetes Configuration File

Generate a kubeconfig file for the kube-proxy service:

```bash
kubectl config set-cluster kubernetes-the-hard-way \
  --certificate-authority=ca.pem \
  --embed-certs=true \
  --server=https://${KUBERNETES_PUBLIC_ADDRESS}:6443 \
  --kubeconfig=kube-proxy.kubeconfig

kubectl config set-credentials system:kube-proxy \
  --client-certificate=kube-proxy.pem \
  --client-key=kube-proxy-key.pem \
  --embed-certs=true \
  --kubeconfig=kube-proxy.kubeconfig

kubectl config set-context default \
  --cluster=kubernetes-the-hard-way \
  --user=system:kube-proxy \
  --kubeconfig=kube-proxy.kubeconfig

kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig
```

This produces:
- `kube-proxy.kubeconfig`

### The kube-controller-manager Kubernetes Configuration File

Generate a kubeconfig file for the kube-controller-manager service. For controllers, use the local loopback interface:

```bash
kubectl config set-cluster kubernetes-the-hard-way \
  --certificate-authority=ca.pem \
  --embed-certs=true \
  --server=https://127.0.0.1:6443 \
  --kubeconfig=kube-controller-manager.kubeconfig

kubectl config set-credentials system:kube-controller-manager \
  --client-certificate=kube-controller-manager.pem \
  --client-key=kube-controller-manager-key.pem \
  --embed-certs=true \
  --kubeconfig=kube-controller-manager.kubeconfig

kubectl config set-context default \
  --cluster=kubernetes-the-hard-way \
  --user=system:kube-controller-manager \
  --kubeconfig=kube-controller-manager.kubeconfig

kubectl config use-context default --kubeconfig=kube-controller-manager.kubeconfig
```

This produces:
- `kube-controller-manager.kubeconfig`

### The kube-scheduler Kubernetes Configuration File

Generate a kubeconfig file for the kube-scheduler service:

```bash
kubectl config set-cluster kubernetes-the-hard-way \
  --certificate-authority=ca.pem \
  --embed-certs=true \
  --server=https://127.0.0.1:6443 \
  --kubeconfig=kube-scheduler.kubeconfig

kubectl config set-credentials system:kube-scheduler \
  --client-certificate=kube-scheduler.pem \
  --client-key=kube-scheduler-key.pem \
  --embed-certs=true \
  --kubeconfig=kube-scheduler.kubeconfig

kubectl config set-context default \
  --cluster=kubernetes-the-hard-way \
  --user=system:kube-scheduler \
  --kubeconfig=kube-scheduler.kubeconfig

kubectl config use-context default --kubeconfig=kube-scheduler.kubeconfig
```

This produces:
- `kube-scheduler.kubeconfig`

### The admin Kubernetes Configuration File

Generate a kubeconfig file for the admin user:

```bash
kubectl config set-cluster kubernetes-the-hard-way \
  --certificate-authority=ca.pem \
  --embed-certs=true \
  --server=https://127.0.0.1:6443 \
  --kubeconfig=admin.kubeconfig

kubectl config set-credentials admin \
  --client-certificate=admin.pem \
  --client-key=admin-key.pem \
  --embed-certs=true \
  --kubeconfig=admin.kubeconfig

kubectl config set-context default \
  --cluster=kubernetes-the-hard-way \
  --user=admin \
  --kubeconfig=admin.kubeconfig

kubectl config use-context default --kubeconfig=admin.kubeconfig
```

This produces:
- `admin.kubeconfig`

## Distribute the Kubernetes Configuration Files

Copy the appropriate kubelet and kube-proxy kubeconfig files to each worker node:

```bash
for instance in worker-214 worker-241 worker-242 worker-243 worker-244; do
  scp ${instance}.kubeconfig kube-proxy.kubeconfig root@${instance}:~/
done
```

Copy the kube-controller-manager, kube-scheduler, and admin kubeconfig files to each controller node:

```bash
for instance in controller-211 controller-212 controller-213; do
  scp admin.kubeconfig kube-controller-manager.kubeconfig kube-scheduler.kubeconfig root@${instance}:~/
done
```

Next: [Generating the Data Encryption Config and Key](06-data-encryption-keys.md)

---

This guide has been updated to reflect your environment:
- **gateway-245**: Public IP 10.10.12.245
- **Controllers**: controller-211, controller-212, controller-213
- **Workers**: worker-214, worker-241, worker-242, worker-243, worker-244
- API Server endpoints for worker clients use the external IP, while controller components use localhost.
- Region details are now US, Pittsburgh, PA.
