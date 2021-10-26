# Bootstrapping the Kubernetes Worker Nodes

In this lab you will bootstrap three Kubernetes worker nodes. The following components will be installed on each node: [runc](https://github.com/opencontainers/runc), [container networking plugins](https://github.com/containernetworking/cni), [containerd](https://github.com/containerd/containerd), [kubelet](https://kubernetes.io/docs/admin/kubelet), and [kube-proxy](https://kubernetes.io/docs/concepts/cluster-administration/proxies).

### What Are the Kubernetes Worker Nodes?
Kubernetes worker nodes are responsible for the actual work of running container applications managed by Kubernetes. “The Kubernetes node has the services necessary to run application containers and be managed from the master systems.” You can find more information about Kubernetes worker nodes in the Kubernetes documentation:

### Kubernetes Worker Node Components
Each Kubernetes worker node consists of the following components
- Kubelet
    - Controls each worker node, providing the APIs that are used by the control plane to manage nodes and pods, and interacts with the container runtime to manage containers
- Kube-proxy
    - Manages iptables rules on the node to provide virtual network access to pods.
- Container runtime
    - Downloads images and runs containers. Two examples of container runtimes are Docker and containerd (Kubernetes the Hard Way uses containerd)
###  Prerequisites
The commands in this lab must be run on each worker instance: `worknode01`, `worknode01`
## Provisioning a Kubernetes Worker Node

Install the OS dependencies:

```
# yum install socat conntrack ipset -y 
```
> The socat binary enables support for the `kubectl port-forward` command.

### Disable Swap

By default the kubelet will fail to start if [swap](https://help.ubuntu.com/community/SwapFaq) is enabled. It is [recommended](https://github.com/kubernetes/kubernetes/issues/7294) that swap be disabled to ensure Kubernetes can provide proper resource allocation and quality of service.

Verify if swap is enabled:

```
sudo swapon --show
```
If output is empthy then swap is not enabled. If swap is enabled run the following command to disable swap immediately:

```
# swapoff -a 
# sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

### Download and Install Worker Binaries

```
# wget https://github.com/kubernetes-sigs/cri-tools/releases/download/v1.21.0/crictl-v1.21.0-linux-amd64.tar.gz \
  https://github.com/opencontainers/runc/releases/download/v1.0.0-rc93/runc.amd64 \
  https://github.com/containernetworking/plugins/releases/download/v0.9.1/cni-plugins-linux-amd64-v0.9.1.tgz \
  https://github.com/containerd/containerd/releases/download/v1.4.4/containerd-1.4.4-linux-amd64.tar.gz \
  https://storage.googleapis.com/kubernetes-release/release/v1.21.0/bin/linux/amd64/kubectl \
  https://storage.googleapis.com/kubernetes-release/release/v1.21.0/bin/linux/amd64/kube-proxy \
  https://storage.googleapis.com/kubernetes-release/release/v1.21.0/bin/linux/amd64/kubelet
```
Create the installation directories:

```
# mkdir -p \
  /etc/cni/net.d \
  /opt/cni/bin \
  /var/lib/kubelet \
  /var/lib/kube-proxy \
  /var/lib/kubernetes \
  /var/run/kubernetes
```

Install the worker binaries:

```
# mkdir containerd
# tar -xvf crictl-v1.21.0-linux-amd64.tar.gz
# tar -xvf containerd-1.4.4-linux-amd64.tar.gz -C containerd
# tar -xvf cni-plugins-linux-amd64-v0.9.1.tgz -C /opt/cni/bin/
# mv runc.amd64 runc
# chmod +x crictl kubectl kube-proxy kubelet runc 
# mv crictl kubectl kube-proxy kubelet runc /usr/local/bin/
# mv containerd/bin/* /bin/
```
### Configure containerd

Create the containerd configuration file:
```
# mkdir -p /etc/containerd/
# cat << EOF | sudo tee /etc/containerd/config.toml
[plugins]
  [plugins.cri.containerd]
    snapshotter = "overlayfs"
    [plugins.cri.containerd.default_runtime]
      runtime_type = "io.containerd.runtime.v1.linux"
      runtime_engine = "/usr/local/bin/runc"
      runtime_root = ""
EOF
```
Create the `containerd.service` systemd unit file:

```
cat <<EOF | sudo tee /etc/systemd/system/containerd.service
[Unit]
Description=containerd container runtime
Documentation=https://containerd.io
After=network.target

[Service]
ExecStartPre=/sbin/modprobe overlay
ExecStart=/bin/containerd
Restart=always
RestartSec=5
Delegate=yes
KillMode=process
OOMScoreAdjust=-999
LimitNOFILE=1048576
LimitNPROC=infinity
LimitCORE=infinity

[Install]
WantedBy=multi-user.target
EOF
```
### Configure the Kubelet

Kubelet is the Kubernetes agent which runs on each worker node. Acting as a middleman between the Kubernetes control plane and the underlying container runtime, it coordinates the running of containers on the worker node. In this lesson, we will configure our systemd service for kubelet. After completing this lesson, you should have a systemd service configured and ready to run on each worker node. You can configure the kubelet service like so. Run these commands on both worker nodes. Set a HOSTNAME environment variable that will be used to generate your config files. Make sure you set the HOSTNAME appropriately for each worker node:

```
# mv ${HOSTNAME}-key.pem ${HOSTNAME}.pem /var/lib/kubelet/
# mv ${HOSTNAME}.kubeconfig /var/lib/kubelet/kubeconfig
# mv ca.pem /var/lib/kubernetes/
```
Create the `kubelet-config.yaml` configuration file:

```
# cat <<EOF | sudo tee /var/lib/kubelet/kubelet-config.yaml
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
authentication:
  anonymous:
    enabled: false
  webhook:
    enabled: true
  x509:
    clientCAFile: "/var/lib/kubernetes/ca.pem"
authorization:
  mode: Webhook
clusterDomain: "cluster.local"
clusterDNS:
  - "10.32.0.10"
runtimeRequestTimeout: "15m"
tlsCertFile: "/var/lib/kubelet/${HOSTNAME}.pem"
tlsPrivateKeyFile: "/var/lib/kubelet/${HOSTNAME}-key.pem"
EOF
```



Create the `kubelet.service` systemd unit file:

```
cat <<EOF | sudo tee /etc/systemd/system/kubelet.service
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/kubernetes/kubernetes
After=containerd.service
Requires=containerd.service

[Service]
ExecStart=/usr/local/bin/kubelet \\
  --config=/var/lib/kubelet/kubelet-config.yaml \\
  --container-runtime=remote \\
  --container-runtime-endpoint=unix:///var/run/containerd/containerd.sock \\
  --image-pull-progress-deadline=2m \\
  --kubeconfig=/var/lib/kubelet/kubeconfig \\
  --network-plugin=cni \\
  --register-node=true \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```
### Configure the Kubernetes Proxy
Kube-proxy is an important component of each Kubernetes worker node. It is responsible for providing network routing to support Kubernetes networking components. In this lesson, we will configure our kube-proxy systemd service. Since this is the last of the three worker node services that we need to configure, we will also go ahead and start all of our worker node services once we're done. Finally, we will complete some steps to verify that our cluster is set up properly and functioning as expected so far. After completing this lesson, you should have two Kubernetes worker nodes up and running, and they should be able to successfully register themselves with the cluster. You can configure the kube-proxy service like so. Run these commands on both worker nodes:
```
# mv kube-proxy.kubeconfig /var/lib/kube-proxy/kubeconfig
```
Create the `kube-proxy-config.yaml` configuration file:
```
# cat <<EOF | sudo tee /var/lib/kube-proxy/kube-proxy-config.yaml
kind: KubeProxyConfiguration
apiVersion: kubeproxy.config.k8s.io/v1alpha1
clientConnection:
  kubeconfig: "/var/lib/kube-proxy/kubeconfig"
mode: "iptables"
clusterCIDR: "10.200.0.0/16"
EOF

```
Create the `kube-proxy.service` systemd unit file:

```
# cat <<EOF | sudo tee /etc/systemd/system/kube-proxy.service
[Unit]
Description=Kubernetes Kube Proxy
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-proxy \\
  --config=/var/lib/kube-proxy/kube-proxy-config.yaml
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```
Start the Worker Services

```
# systemctl daemon-reload
# systemctl enable containerd kubelet kube-proxy
# systemctl start containerd kubelet kube-proxy
```
## Verification
Finally, verify that both workers have registered themselves with the cluster. Log in to one of your control nodes and run this: 
First should we create a dir in both of controller nodes 
will crate dir for kubectl to containe the certificate and config 

```
# mkdir -p $HOME/.kube
# cp -i admin.kubeconfig $HOME/.kube/config
# chown $(id -u):$(id -g) $HOME/.kube/config
# kubectl get nodes
```
> output

```
NAME                 STATUS     ROLES    AGE     VERSION
worknode01.k8s.com   NotReady   <none>   5m28s   v1.21.0
worknode02.k8s.com   NotReady   <none>   5m31s   v1.21.0
```
> dotnot wary about `NotReady` because in networking will fix this issues 

Next: [Configuring kubectl for Remote Access](09-configuring-kubectl.md)