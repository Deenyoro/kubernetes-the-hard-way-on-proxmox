Below is the updated guide for generating the data encryption configuration and key, customized for your environment. In your setup, the controller nodes are named **controller-211**, **controller-212**, and **controller-213**.

# Generating the Data Encryption Config and Key

Kubernetes stores a variety of data—including cluster state, application configurations, and secrets—and supports the ability to [encrypt](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data) data at rest.

In this lab you will generate an encryption key and create an [encryption config](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/#understanding-the-encryption-at-rest-configuration) suitable for encrypting Kubernetes Secrets.

## The Encryption Key

Generate an encryption key:

```bash
ENCRYPTION_KEY=$(head -c 32 /dev/urandom | base64)
```

## The Encryption Config File

Create the `encryption-config.yaml` file:

```bash
cat > encryption-config.yaml <<EOF
kind: EncryptionConfig
apiVersion: v1
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: ${ENCRYPTION_KEY}
      - identity: {}
EOF
```

## Distribute the Encryption Config

Copy the `encryption-config.yaml` file to each controller node:

```bash
for instance in controller-211 controller-212 controller-213; do
  scp encryption-config.yaml root@${instance}:~/
done
```

Next: [Bootstrapping the etcd Cluster](07-bootstrapping-etcd.md)

This guide reflects your environment, where your controller nodes are **controller-211**, **controller-212**, and **controller-213**.
