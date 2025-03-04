# Smoke Test

In this lab you will perform a series of tests to ensure your Kubernetes cluster is functioning correctly in your environment.

## Data Encryption

Verify that secret data is encrypted at rest.

1. **Create a generic secret:**

   ```bash
   kubectl create secret generic kubernetes-the-hard-way \
     --from-literal="mykey=mydata"
   ```

2. **Print a hexdump of the secret stored in etcd:**

   Run this command on one of your controller nodes (for example, on **controller-211**):

   ```bash
   ssh root@controller-211 \
     sudo ETCDCTL_API=3 etcdctl get \
       --endpoints=https://127.0.0.1:2379 \
       --cacert=/etc/etcd/ca.pem \
       --cert=/etc/etcd/kubernetes.pem \
       --key=/etc/etcd/kubernetes-key.pem \
       /registry/secrets/default/kubernetes-the-hard-way | hexdump -C
   ```

   The output should begin with a prefix like `k8s:enc:aescbc:v1:key1`, indicating that the data was encrypted using the `aescbc` provider with the key named `key1`.

## Deployments

Verify that you can create and manage deployments.

1. **Create an nginx deployment:**

   ```bash
   kubectl create deployment nginx --image=nginx
   ```

2. **List the pods created by the nginx deployment:**

   ```bash
   kubectl get pods -l app=nginx
   ```

   You should see output similar to:

   ```plaintext
   NAME                     READY   STATUS    RESTARTS   AGE
   nginx-554b9c67f9-vt5rn   1/1     Running   0          10s
   ```

### Port Forwarding

1. **Retrieve the pod name:**

   ```bash
   POD_NAME=$(kubectl get pods -l app=nginx -o jsonpath="{.items[0].metadata.name}")
   ```

2. **Forward port 8080 on your local machine to port 80 on the nginx pod:**

   ```bash
   kubectl port-forward $POD_NAME 8080:80
   ```

   You should see output like:

   ```plaintext
   Forwarding from 127.0.0.1:8080 -> 80
   Forwarding from [::1]:8080 -> 80
   ```

3. **In a new terminal, verify the forwarding by making an HTTP HEAD request:**

   ```bash
   curl --head http://127.0.0.1:8080
   ```

   Expected output includes `HTTP/1.1 200 OK`.

   When finished, press `Ctrl+C` to stop port forwarding.

### Logs

Check the logs of the nginx pod:

```bash
kubectl logs $POD_NAME
```

You should see log entries similar to:

```plaintext
127.0.0.1 - - [24/Jun/2020:12:55:15 +0000] "HEAD / HTTP/1.1" 200 0 "-" "curl/7.64.0" "-"
```

### Exec

Verify command execution inside the container by printing the nginx version:

```bash
kubectl exec -ti $POD_NAME -- nginx -v
```

Expected output:

```plaintext
nginx version: nginx/1.19.0
```

## Services

Expose the nginx deployment via a NodePort service.

1. **Expose the deployment:**

   ```bash
   kubectl expose deployment nginx --port 80 --type NodePort
   ```

2. **Retrieve the assigned node port:**

   ```bash
   NODE_PORT=$(kubectl get svc nginx --output=jsonpath='{range .spec.ports[0]}{.nodePort}')
   ```

3. **Define the IP of a worker node for testing.** For example, using **worker-214**:

   ```bash
   NODE_IP=192.168.1.214
   ```

4. **Make an HTTP HEAD request to the nginx service via the worker node:**

   ```bash
   curl -I http://${NODE_IP}:${NODE_PORT}
   ```

   Expected output should include:

   ```plaintext
   HTTP/1.1 200 OK
   Server: nginx/1.19.0
   ...
   ```

Next: [Cleaning Up](14-cleanup.md)

---

This guide uses your environment details:
- **Gateway VM (gateway-245)** has public IP **10.10.12.245**.
- Controller nodes: **controller-211**, **controller-212**, **controller-213**.
- Worker nodes: **worker-214**, **worker-241**, **worker-242**, **worker-243**, **worker-244**.
- Region: Pittsburgh, PA.

You can now proceed with the smoke test to verify that your cluster is functioning correctly.
