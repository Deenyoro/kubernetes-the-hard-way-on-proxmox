Below is the revised guide for your setup. In your environment:

- Your public network uses a default gateway of **10.10.12.1**.
- Your gateway VM (now named **gateway-245**) has a public IP of **10.10.12.245/24** and a private IP of **192.168.1.245/24**.
- Your Kubernetes nodes use IPs in the **192.168.1.0/24** range.
- (For pods, we’ll keep the 10.200.0.0/16 range; adjust the subnets if you have more than three workers.)

---

# Installing the Client Tools

In this lab you will install the command line utilities required to complete this tutorial: [cfssl](https://github.com/cloudflare/cfssl), [cfssljson](https://github.com/cloudflare/cfssl), and [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl).

## Install CFSSL

The `cfssl` and `cfssljson` utilities are used to provision a [PKI Infrastructure](https://en.wikipedia.org/wiki/Public_key_infrastructure) and generate TLS certificates.

On the **gateway-245** VM, download and install `cfssl` and `cfssljson`:

```bash
wget -q --show-progress --https-only --timestamping https://github.com/cloudflare/cfssl/releases/download/v1.6.4/cfssl_1.6.4_linux_amd64 -O cfssl
wget -q --show-progress --https-only --timestamping https://github.com/cloudflare/cfssl/releases/download/v1.6.4/cfssljson_1.6.4_linux_amd64 -O cfssljson
```

Make them executable:

```bash
chmod +x cfssl cfssljson
```

Then move them to your PATH:

```bash
sudo mv cfssl cfssljson /usr/local/bin/
```

### Verification

Verify that `cfssl` and `cfssljson` (version 1.3.4 or higher) are installed:

```bash
cfssl version
```

> Expected output:
>
> ```bash
> Version: 1.3.4
> Revision: dev
> Runtime: go1.13
> ```

And:

```bash
cfssljson --version
```

> Expected output:
>
> ```bash
> Version: 1.3.4
> Revision: dev
> Runtime: go1.13
> ```

## Install kubectl

The `kubectl` tool lets you interact with the Kubernetes API Server. On the **gateway-245** VM, download and install `kubectl` from the official release binaries:

```bash
wget https://storage.googleapis.com/kubernetes-release/release/v1.29.1/bin/linux/amd64/kubectl
```

Make it executable:

```bash
chmod +x kubectl
```

Then move it into your PATH:

```bash
sudo mv kubectl /usr/local/bin/
```

### Verification

Confirm that `kubectl` (version 1.15.3 or higher) is installed:

```bash
kubectl version --client
```

> Expected output:
>
> ```bash
> Client Version: v1.29.1
> Kustomize Version: v5.0.4-0.20230601165947-6ce0bf390ce3
> ```

Next: [Provisioning Compute Resources](03-compute-resources.md)
