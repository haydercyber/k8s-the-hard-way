# Installing the Client Tools

In this lab you will install the command line utilities required to complete this tutorial: [cfssl](https://github.com/cloudflare/cfssl), [cfssljson](https://github.com/cloudflare/cfssl), and [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl).
in this lissions will work on remote kubectl 


## Install CFSSL

The `cfssl` and `cfssljson` command line utilities will be used to provision a [PKI Infrastructure](https://en.wikipedia.org/wiki/Public_key_infrastructure) and generate TLS certificates.

Download and install `cfssl` and `cfssljson`:
```
# wget https://storage.googleapis.com/kubernetes-the-hard-way/cfssl/1.4.1/linux/cfssl
# wget https://storage.googleapis.com/kubernetes-the-hard-way/cfssl/1.4.1/linux/cfssljson
# chmod +x cfssl cfssljson
# sudo mv cfssl cfssljson /usr/local/bin/
```

## Verification

Verify `cfssl` and `cfssljson` version 1.4.1 or higher is installed:

```
cfssl version
```

> output

```
Version: 1.4.1
Runtime: go1.12.12
```

```
cfssljson --version
```
```
Version: 1.4.1
Runtime: go1.12.12
```
## Install kubectl

The `kubectl` command line utility is used to interact with the Kubernetes API Server. Download and install `kubectl` from the official release binaries:

```
# wget https://storage.googleapis.com/kubernetes-release/release/v1.21.0/bin/linux/amd64/kubectl
# chmod +x kubectl
# sudo mv kubectl /usr/local/bin/
```
### Verification

Verify `kubectl` version 1.21.0 or higher is installed:

```
kubectl version --client
```

> output

```
Client Version: version.Info{Major:"1", Minor:"21", GitVersion:"v1.21.0", GitCommit:"cb303e613a121a29364f75cc67d3d580833a7479", GitTreeState:"clean", BuildDate:"2021-04-08T16:31:21Z", GoVersion:"go1.16.1", Compiler:"gc", Platform:"linux/amd64"}
```

Next: [Provisioning the CA and Generating TLS Certificates](03-certificate-authority.md)