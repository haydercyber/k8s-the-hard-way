# Generating Kubernetes Configuration Files for Authentication

In this lab you will generate [Kubernetes configuration files](https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/), also known as kubeconfigs, which enable Kubernetes clients to locate and authenticate to the Kubernetes API Servers.
## What Are Kubeconfigs and Why Do We Need Them?
- Kubeconfigs
    - A Kubernetes configuration file, or kubeconfig, is a file that stores “information about clusters, users, namespaces, and authentication     mechanisms.” It contains the configuration data needed to connect to and interact with one or more Kubernetes clusters. You can find more information about kubeconfigs in the Kubernetes documentation: [Kubernetes configuration files](https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/), Kubeconfigs contain information such as
        - The location of the cluster you want to connect to
        - What user you want to authenticate as
        - Data needed in order to authenticate, such as tokens or client certificates
    - You can even define multiple contexts in a kubeconfig file, allowing you to easily switch between multiple clusters.
## Why Do We Need Kubeconfigs? 
- How to Generate a Kubeconfig 
    - Kubeconfigs can be generated using kubectl
```
# kubectl config set-cluster // set up the configuration for the location of the cluster.
```
        
```
# kubectl config set-credentials // set the username and client certificate that will be used to authenticate.
```
 
```
# kubectl config set-context default // set up the default context
```

```
# kubectl config use-context default // set the current context to the configuration we provided
```

## What Kubeconfigs Do We Need to Generate? 
- We will need several Kubeconfig files for various components of the Kubernetes cluster:
    - Kubelet(one for each worker node)
    - Kube-proxy
    - Kube-controller-manager
    - Kube-scheduler 
    - Admin
- The next step in building a Kubernetes cluster the hard way is to generate kubeconfigs which will be used by the various services that will make up the cluster. In this lesson, we will generate these kubeconfigs. After completing this lesson, you should have a set of kubeconfigs which you will need later in order to configure the Kubernetes cluster. Here are the commands used in the demo. Be sure to replace the placeholders with actual values from your machine . Create an environment variable to store the address of the Kubernetes API, and set it to the  IP of your load balancer
> in our digram the ip for loadblancer is 192.168.0.3 you can see blow 
![digram screenshot](images/k8s_digram)
```
# KUBERNETES_PUBLIC_ADDRESS=192.168.0.3
```
## Client Authentication Configs

In this section you will generate kubeconfig files for the `controller manager`, `kubelet`, `kube-proxy`, and `scheduler` clients and the `admin` user.
### The kubelet Kubernetes Configuration File

When generating kubeconfig files for Kubelets the client certificate matching the Kubelet's node name must be used. This will ensure Kubelets are properly authorized by the Kubernetes [Node Authorizer](https://kubernetes.io/docs/admin/authorization/node/).

> The following commands must be run in the same directory used to generate the SSL certificates during the [Generating TLS Certificates](04-certificate-authority.md) lab.

Generate a kubeconfig file for each worker node:

```
for instance in worknode01.k8s.com worknode02.k8s.com; do
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
Results:

```
worknode01.k8s.com.kubeconfig
worknode02.k8s.com.kubeconfig
```
### The kube-proxy Kubernetes Configuration File

Generate a kubeconfig file for the `kube-proxy` service:

```
{
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
}
```
Results:

```
kube-proxy.kubeconfig
```
### The kube-controller-manager Kubernetes Configuration File

Generate a kubeconfig file for the `kube-controller-manager` service:

```
{
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
}
```
Results:

```
kube-controller-manager.kubeconfig
```


### The kube-scheduler Kubernetes Configuration File

Generate a kubeconfig file for the `kube-scheduler` service:

```
{
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
}
```

Results:

```
kube-scheduler.kubeconfig
```
### The admin Kubernetes Configuration File

Generate a kubeconfig file for the `admin` user:

```
{
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
}
```

Results:

```
admin.kubeconfig
```
## Distribute the Kubernetes Configuration Files

Copy the appropriate `kubelet` and `kube-proxy` kubeconfig files to each worker instance:

```
for instance in worknode01.k8s.com worknode02.k8s.com; do
    scp ${instance}.kubeconfig kube-proxy.kubeconfig root@${instance}:~/
done
```
Copy the appropriate `kube-controller-manager` and `kube-scheduler` kubeconfig files to each controller instance:

```
for instance in kubecon01.k8s.com kubecon02.k8s.com; do 
    scp admin.kubeconfig kube-controller-manager.kubeconfig kube-scheduler.kubeconfig root@${instance}:~/
done
```
Next: [Generating the Data Encryption Config and Key](05-data-encryption-keys.md)
