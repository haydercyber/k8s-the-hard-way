# Configuring kubectl for Remote Access

In this lab you will generate a kubeconfig file for the `kubectl` command line utility based on the `admin` user credentials.

### What Is Kubectl? 
Kubectl
- is the Kubernetes command line tool. It allows us to interact with Kubernetes clusters from the command line.
We will set up kubectl to allow remote access from our machine in order to manage the cluster remotely. To do this, we will generate a local kubeconfig that will authenticate as the admin user and access the Kubernetes API through the load balancer.
In this lab you will generate a kubeconfig file for the kubectl command line utility based on the admin user credentials.
Run the commands in this lab from the same directory used to generate the admin client certificates.
There are a few steps to configuring a local kubectl installation for managing a remote cluster. This lesson will guide you through that process. After completing this lesson, you should have a local kubectl installation that is capable of running kubectl commands against your remote Kubernetes cluster. In a separate shell, open up an ssh tunnel to port 6443 on your Kubernetes API load balancer: 
The Admin Kubernetes Configuration File
Each kubeconfig requires a Kubernetes API Server to connect to. To support high availability the IP address assigned to the external load balancer fronting the Kubernetes API Servers will be used. 
Generate a kubeconfig file suitable for authenticating as the admin user:
Lets configure the
```
# cd /k8s
# mkdir -p $HOME/.kube
# cp -i admin.kubeconfig $HOME/.kube/config
# chown $(id -u):$(id -g) $HOME/.kube/config
# KUBERNETES_PUBLIC_ADDRESS=192.168.0.3
```
## The Admin Kubernetes Configuration File

Each kubeconfig requires a Kubernetes API Server to connect to. To support high availability the IP address assigned to the external load balancer fronting the Kubernetes API Servers will be used.

Generate a kubeconfig file suitable for authenticating as the `admin` user:

```
{
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
}
```
## Verification

Check the version of the remote Kubernetes cluster:

```
kubectl version
```

> output

```
Client Version: version.Info{Major:"1", Minor:"21", GitVersion:"v1.21.0", GitCommit:"cb303e613a121a29364f75cc67d3d580833a7479", GitTreeState:"clean", BuildDate:"2021-04-08T16:31:21Z", GoVersion:"go1.16.1", Compiler:"gc", Platform:"linux/amd64"}
Server Version: version.Info{Major:"1", Minor:"21", GitVersion:"v1.21.0", GitCommit:"cb303e613a121a29364f75cc67d3d580833a7479", GitTreeState:"clean", BuildDate:"2021-04-08T16:25:06Z", GoVersion:"go1.16.1", Compiler:"gc", Platform:"linux/amd64"}
```

List the nodes in the remote Kubernetes cluster:

```
kubectl get nodes
```
> output

```
NAME                 STATUS     ROLES    AGE     VERSION
worknode01.k8s.com   NotReady   <none>   5m28s   v1.21.0
worknode02.k8s.com   NotReady   <none>   5m31s   v1.21.0
```
> dotnot wary about `NotReady` because in networking will fix this issues 
Next: [Provisioning Pod Network Routes](10-pod-network-routes.md)