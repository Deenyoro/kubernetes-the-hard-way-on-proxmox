Below is the revised guide for configuring **kubectl** for remote access in your Pittsburgh, PA environment. In this setup, you will generate a kubeconfig file for the **admin** user. The external load balancer is provided by your **gateway-245** VM, which has a public IP of **10.10.12.245**.

Run these commands from the directory where you generated your admin client certificates.

---

# Configuring kubectl for Remote Access

In this lab you will generate a kubeconfig file for the `kubectl` command line utility based on the `admin` user credentials.

## The Admin Kubernetes Configuration File

Each kubeconfig requires a Kubernetes API Server endpoint. In our environment the external load balancer on **gateway-245** provides this endpoint. Set the public address as follows:

```bash
KUBERNETES_PUBLIC_ADDRESS=10.10.12.245
```

Generate a kubeconfig file for the `admin` user:

```bash
kubectl config set-cluster kubernetes-the-hard-way \
  --certificate-authority=ca.pem \
  --embed-certs=true \
  --server=https://${KUBERNETES_PUBLIC_ADDRESS}:6443

kubectl config set-credentials admin \
  --client-certificate=admin.pem \
  --client-key=admin-key.pem

kubectl config set-context kubernetes-the-hard-way \
  --cluster=kubernetes-the-hard-way \
  --user=admin

kubectl config use-context kubernetes-the-hard-way
```

## Verification

To verify remote access to your Kubernetes cluster, check the component statuses:

```bash
kubectl get --raw='/readyz?verbose'
```

Expected output should confirm that all components (etcd, controller-manager, scheduler, etc.) are healthy.

You can also list the nodes registered in your cluster:

```bash
kubectl get nodes
```

Expected output might look similar to:

```bash
NAME           STATUS   ROLES    AGE   VERSION
worker-214     Ready    <none>   5m    v1.29.1
worker-241     Ready    <none>   5m    v1.29.1
worker-242     Ready    <none>   5m    v1.29.1
worker-243     Ready    <none>   5m    v1.29.1
worker-244     Ready    <none>   5m    v1.29.1
```

Next: [Provisioning Pod Network Routes](11-pod-network-routes.md)


This guide uses your environment details:  
- **Gateway VM (gateway-245)**: Public IP **10.10.12.245**  
- Controllers: **controller-211**, **controller-212**, **controller-213**  
- Workers: **worker-214**, **worker-241**, **worker-242**, **worker-243**, **worker-244**  
- Region: Pittsburgh, PA

You can now use **kubectl** with the generated configuration to manage your remote cluster.
