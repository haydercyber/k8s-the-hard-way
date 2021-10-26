# Generating the Data Encryption Config and Key

Kubernetes stores a variety of data including cluster state, application configurations, and secrets. Kubernetes supports the ability to [encrypt](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data) cluster data at rest.

In this lab you will generate an encryption key and an [encryption config](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/#understanding-the-encryption-at-rest-configuration) suitable for encrypting Kubernetes Secrets.

## What Ist he Kubernetes Data Encryption Config?
- Kubernetes Secret Encryption 
    - Kubernetes supports the ability to encrypt secret data at rest. This means that secrets are encrypted so that they are never stored on disc in plain text. This feature is important for security, but in order to use it we need to provide Kubernetes with an encryption key. We will generate an encryption key and put it into a configuration file. We will then copy that file to our Kubernetes controller servers.


    - In order to make use of Kubernetes' ability to encrypt sensitive data at rest, you need to provide Kubernetes with an encrpytion key using a data encryption config file. This lesson walks you through the process of creating a encryption key and storing it in the necessary file, as well as showing how to copy that file to your Kubernetes controllers. After completing this lesson, you should have a valid Kubernetes data encryption config file, and there should be a copy of that file on each of your Kubernetes controller servers. 
## The Encryption Key
Here are the commands used in the demo. Generate the Kubernetes Data encrpytion config file containing the encrpytion key

Generate an encryption key:

```
ENCRYPTION_KEY=$(head -c 32 /dev/urandom | base64)
```
## The Encryption Config File

Create the `encryption-config.yaml` encryption config file:

```
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
## Distribute the Kubernetes Encryption Config
Copy the `encryption-config.yaml` encryption config file to each controller instance:

```
for instance in kubecon01.k8s.com kubecon01.k8s.com; do
     scp encryption-config.yaml ${instance}:~/
done
```
Next: [Bootstrapping the etcd Cluster](06-bootstrapping-etcd.md)
