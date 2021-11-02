# Kubernetes The Hard Way on Centos
This tutorial walks you through setting up Kubernetes the hard way. This guide is not for people looking for a fully automated command to bring up a Kubernetes cluster. If that's you then check out [Google Kubernetes Engine](https://cloud.google.com/kubernetes-engine), or the [Getting Started Guides](https://kubernetes.io/docs/setup).

Kubernetes The Hard Way is optimized for learning, which means taking the long route to ensure you understand each task required to bootstrap a Kubernetes cluster.

> The results of this tutorial should not be viewed as production ready, and may receive limited support from the community, but don't let that stop you from learning!

## Target Audience

The target audience for this tutorial is someone planning to support a production Kubernetes cluster and wants to understand how everything fits together.

## Cluster Details

Kubernetes The Hard Way guides you through bootstrapping a highly available Kubernetes cluster with end-to-end encryption between components and RBAC authentication.

* [kubernetes](https://github.com/kubernetes/kubernetes) v1.21.0
* [containerd](https://github.com/containerd/containerd) v1.4.4
* [coredns](https://github.com/coredns/coredns) v1.8.3
* [cni](https://github.com/containernetworking/cni) v0.9.1
* [etcd](https://github.com/etcd-io/etcd) v3.4.15

## Labs

This tutorial install on centos 7.9 the Offical docoumnt have some error for deployment on centos because it install on GCP 

* [Prerequisites](docs/01-prerequisites.md)
* [Installing the Client Tools](docs/02-client-tools.md)
* [Provisioning the CA and Generating TLS Certificates](docs/03-certificate-authority.md)
* [Generating Kubernetes Configuration Files for Authentication](docs/04-kubernetes-configuration-files.md)
* [Generating the Data Encryption Config and Key](docs/05-data-encryption-keys.md)
* [Bootstrapping the etcd Cluster](docs/06-bootstrapping-etcd.md)
* [Bootstrapping the Kubernetes Control Plane](docs/07-bootstrapping-kubernetes-controllers.md)
* [Bootstrapping the Kubernetes Worker Nodes](docs/08-bootstrapping-kubernetes-workers.md)
* [Configuring kubectl for Remote Access](docs/09-configuring-kubectl.md)
* [Provisioning Pod Network Routes](docs/10-pod-network-routes.md)
* [Deploying the DNS Cluster Add-on](docs/11-dns-addon.md)
* [Smoke Test](docs/12-smoke-test.md)
* [Cleaning Up](docs/13-cleanup.md)
