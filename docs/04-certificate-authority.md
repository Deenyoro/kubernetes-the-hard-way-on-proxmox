# Provisioning a CA and Generating TLS Certificates

In this lab you will provision a [PKI Infrastructure](https://en.wikipedia.org/wiki/Public_key_infrastructure) using CloudFlare's PKI toolkit, [cfssl](https://github.com/cloudflare/cfssl). You will bootstrap a Certificate Authority (CA) and generate TLS certificates for etcd, kube-apiserver, kube-controller-manager, kube-scheduler, kubelet, kube-proxy, and the Kubernetes admin user.

> **Note:** Run all commands on the **gateway-245** VM.

## Certificate Authority

Create the CA configuration file (`ca-config.json`):

```bash
cat > ca-config.json <<EOF
{
  "signing": {
    "default": {
      "expiry": "8760h"
    },
    "profiles": {
      "kubernetes": {
        "usages": ["signing", "key encipherment", "server auth", "client auth"],
        "expiry": "8760h"
      }
    }
  }
}
EOF
```

Create the CA certificate signing request (`ca-csr.json`):

```bash
cat > ca-csr.json <<EOF
{
  "CN": "Kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Pittsburgh",
      "O": "Kubernetes",
      "OU": "CA",
      "ST": "PA"
    }
  ]
}
EOF
```

Generate the CA certificate and key:

```bash
cfssl gencert -initca ca-csr.json | cfssljson -bare ca
```

This produces the files:  
• `ca.pem`  
• `ca-key.pem`

## Client and Server Certificates

### The Admin Client Certificate

Generate the admin client certificate and private key:

```bash
cat > admin-csr.json <<EOF
{
  "CN": "admin",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Pittsburgh",
      "O": "system:masters",
      "OU": "Kubernetes The Hard Way",
      "ST": "PA"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  admin-csr.json | cfssljson -bare admin
```

This produces:  
• `admin.pem`  
• `admin-key.pem`

### The Kubelet (Worker Node) Client Certificates

Kubelets must identify themselves as members of the `system:nodes` group (with username `system:node:<nodeName>`). In your environment you have five worker nodes. First, set your gateway’s public IP:

```bash
GATEWAY_PUBLIC_IP=10.10.12.245
```

Then, for each worker node, generate a certificate. For example, using a Bash loop with an associative array:

```bash
declare -A WORKERS=(
  ["worker-214"]="192.168.1.214"
  ["worker-241"]="192.168.1.241"
  ["worker-242"]="192.168.1.242"
  ["worker-243"]="192.168.1.243"
  ["worker-244"]="192.168.1.244"
)

for worker in "${!WORKERS[@]}"; do
  cat > ${worker}-csr.json <<EOF
{
  "CN": "system:node:${worker}",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Pittsburgh",
      "O": "system:nodes",
      "OU": "Kubernetes The Hard Way",
      "ST": "PA"
    }
  ]
}
EOF

  cfssl gencert \
    -ca=ca.pem \
    -ca-key=ca-key.pem \
    -config=ca-config.json \
    -hostname=${worker},${GATEWAY_PUBLIC_IP},${WORKERS[$worker]} \
    -profile=kubernetes \
    ${worker}-csr.json | cfssljson -bare ${worker}
done
```

This produces for each worker:  
• `<worker>.pem` and `<worker>-key.pem` (for example, `worker-214.pem` and `worker-214-key.pem`).

### The Controller Manager Client Certificate

Generate the kube-controller-manager certificate and key:

```bash
cat > kube-controller-manager-csr.json <<EOF
{
  "CN": "system:kube-controller-manager",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Pittsburgh",
      "O": "system:kube-controller-manager",
      "OU": "Kubernetes The Hard Way",
      "ST": "PA"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-controller-manager-csr.json | cfssljson -bare kube-controller-manager
```

This produces:  
• `kube-controller-manager.pem`  
• `kube-controller-manager-key.pem`

### The Kube Proxy Client Certificate

Generate the kube-proxy certificate and key:

```bash
cat > kube-proxy-csr.json <<EOF
{
  "CN": "system:kube-proxy",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Pittsburgh",
      "O": "system:node-proxier",
      "OU": "Kubernetes The Hard Way",
      "ST": "PA"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-proxy-csr.json | cfssljson -bare kube-proxy
```

This produces:  
• `kube-proxy.pem`  
• `kube-proxy-key.pem`

### The Scheduler Client Certificate

Generate the kube-scheduler certificate and key:

```bash
cat > kube-scheduler-csr.json <<EOF
{
  "CN": "system:kube-scheduler",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Pittsburgh",
      "O": "system:kube-scheduler",
      "OU": "Kubernetes The Hard Way",
      "ST": "PA"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-scheduler-csr.json | cfssljson -bare kube-scheduler
```

This produces:  
• `kube-scheduler.pem`  
• `kube-scheduler-key.pem`

### The Kubernetes API Server Certificate

The API server certificate must cover both the internal service IP and all controller node IPs, as well as the public address. In your environment, assume:  
- The service IP remains **10.32.0.1**  
- The controllers are at **192.168.1.211**, **192.168.1.212**, and **192.168.1.213**  
- The public IP (used by remote clients) is **10.10.12.245**

Also include the standard DNS names for Kubernetes:

```bash
KUBERNETES_PUBLIC_ADDRESS=10.10.12.245
KUBERNETES_HOSTNAMES="kubernetes,kubernetes.default,kubernetes.default.svc,kubernetes.svc.cluster.local"

cat > kubernetes-csr.json <<EOF
{
  "CN": "kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Pittsburgh",
      "O": "Kubernetes",
      "OU": "Kubernetes The Hard Way",
      "ST": "PA"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -hostname=10.32.0.1,192.168.1.211,192.168.1.212,192.168.1.213,${KUBERNETES_PUBLIC_ADDRESS},127.0.0.1,${KUBERNETES_HOSTNAMES} \
  -profile=kubernetes \
  kubernetes-csr.json | cfssljson -bare kubernetes
```

This produces:  
• `kubernetes.pem`  
• `kubernetes-key.pem`

## The Service Account Key Pair

Generate a key pair for signing service account tokens:

```bash
cat > service-account-csr.json <<EOF
{
  "CN": "service-accounts",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Pittsburgh",
      "O": "Kubernetes",
      "OU": "Kubernetes The Hard Way",
      "ST": "PA"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  service-account-csr.json | cfssljson -bare service-account
```

This produces:  
• `service-account.pem`  
• `service-account-key.pem`

## Distribute the Client and Server Certificates

### To Worker Nodes

Copy the following files to each worker node (in your case: **worker-214**, **worker-241**, **worker-242**, **worker-243**, **worker-244**):

```bash
for instance in worker-214 worker-241 worker-242 worker-243 worker-244; do
  scp ca.pem ${instance}-key.pem ${instance}.pem root@${instance}:~/
done
```

### To Controller Nodes

Copy the following files to each controller node (**controller-211**, **controller-212**, **controller-213**):

```bash
for instance in controller-211 controller-212 controller-213; do
  scp ca.pem ca-key.pem kubernetes-key.pem kubernetes.pem \
    service-account-key.pem service-account.pem root@${instance}:~/
done
```

> In later labs, the `kube-proxy`, `kube-controller-manager`, `kube-scheduler`, and `kubelet` client certificates will be used to generate client authentication configuration files.

Next: [Generating Kubernetes Configuration Files for Authentication](05-kubernetes-configuration-files.md)


---

This updated guide now reflects your environment with:  
- **gateway-245**: Public IP **10.10.12.245**; Private IP **192.168.1.245**  
- **Controllers**: **192.168.1.211**, **192.168.1.212**, **192.168.1.213**  
- **Workers**: worker-214 (192.168.1.214), worker-241 (192.168.1.241), worker-242 (192.168.1.242), worker-243 (192.168.1.243), worker-244 (192.168.1.244)  
- Region details updated to **US**, **Pittsburgh**, **PA**
