# Bootstrapping the Kubernetes Control Plane

In this lab you will bootstrap the Kubernetes control plane across three compute instances and configure it for high availability. You will also create an external load balancer that exposes the Kubernetes API Servers to remote clients. The following components will be installed on each node: Kubernetes API Server, Scheduler, and Controller Manager.

## Prerequisites

The commands in this lab must be run on each controller instance: `kubecon01`, `kubecon02`

## Provision the Kubernetes Control Plane

The first step in bootstrapping a new Kubernetes control plane is to install the necessary binaries on the controller servers. We will walk through the process of downloading and installing the binaries on both Kubernetes controllers. This will prepare your environment for the lessons that follow, in which we will configure these binaries to run as  systemd  services. You can install the control plane binaries on each control node like this
```
# mkdir -p /etc/kubernetes/config
```
### Download and Install the Kubernetes Controller Binaries

Download the official Kubernetes release binaries

```
# wget "https://storage.googleapis.com/kubernetes-release/release/v1.21.0/bin/linux/amd64/kube-apiserver" \
  "https://storage.googleapis.com/kubernetes-release/release/v1.21.0/bin/linux/amd64/kube-controller-manager" \
  "https://storage.googleapis.com/kubernetes-release/release/v1.21.0/bin/linux/amd64/kube-scheduler" \
  "https://storage.googleapis.com/kubernetes-release/release/v1.21.0/bin/linux/amd64/kubectl"
```
Lets change the permission to be executable 

```
# chmod +x kube-apiserver kube-controller-manager kube-scheduler kubectl 
```
Lets mv binary to /usr/local/bin 

```
# mv kube-apiserver kube-controller-manager kube-scheduler kubectl /usr/local/bin/
```
### Configure the Kubernetes API Server
The Kubernetes API server provides the primary interface for the Kubernetes control plane and the cluster as a whole. When you interact with Kubernetes, you are nearly always doing it through the Kubernetes API server. This lesson will guide you through the process of configuring the kube-apiserver service on your two Kubernetes control nodes. After completing this lesson, you should have a  systemd  unit set up to run kube-apiserver as a service on each Kubernetes control node. You can configure the Kubernetes API server like so

```
# mkdir -p /var/lib/kubernetes/
# mv ca.pem ca-key.pem kubernetes-key.pem kubernetes.pem \
    service-account-key.pem service-account.pem \
    encryption-config.yaml /var/lib/kubernetes/
```
Set some environment variables that will be used to create the  systemd  unit file. Make sure you replace the placeholders with their actual values
```
# INTERNAL_IP=$(/sbin/ip -o -4 addr list eth0 | awk '{print $4}' | cut -d/ -f1)
# CONTROLLER0_IP=192.168.0.1
# KUBERNETES_PUBLIC_ADDRESS=$(/sbin/ip -o -4 addr list eth0 | awk '{print $4}' | cut -d/ -f1)
# CONTROLLER1_IP=192.168.0.2
```
Generate the kube-apiserver unit file for  systemd : 

```
# cat <<EOF | sudo tee /etc/systemd/system/kube-apiserver.service
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-apiserver \\
  --advertise-address=${INTERNAL_IP} \\
  --allow-privileged=true \\
  --apiserver-count=3 \\
  --audit-log-maxage=30 \\
  --audit-log-maxbackup=3 \\
  --audit-log-maxsize=100 \\
  --audit-log-path=/var/log/audit.log \\
  --authorization-mode=Node,RBAC \\
  --bind-address=0.0.0.0 \\
  --client-ca-file=/var/lib/kubernetes/ca.pem \\
  --enable-admission-plugins=NamespaceLifecycle,NodeRestriction,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota \\
  --etcd-cafile=/var/lib/kubernetes/ca.pem \\
  --etcd-certfile=/var/lib/kubernetes/kubernetes.pem \\
  --etcd-keyfile=/var/lib/kubernetes/kubernetes-key.pem \\
  --etcd-servers=https://${CONTROLLER0_IP}:2379,https://${CONTROLLER1_IP}:2379 \\
  --event-ttl=1h \\
  --encryption-provider-config=/var/lib/kubernetes/encryption-config.yaml \\
  --kubelet-certificate-authority=/var/lib/kubernetes/ca.pem \\
  --kubelet-client-certificate=/var/lib/kubernetes/kubernetes.pem \\
  --kubelet-client-key=/var/lib/kubernetes/kubernetes-key.pem \\
  --service-account-key-file=/var/lib/kubernetes/service-account.pem \\
  --service-account-signing-key-file=/var/lib/kubernetes/service-account-key.pem \\
  --service-account-issuer=https://${KUBERNETES_PUBLIC_ADDRESS}:6443 \\
  --service-cluster-ip-range=10.32.0.0/24 \\
  --service-node-port-range=30000-32767 \\
  --tls-cert-file=/var/lib/kubernetes/kubernetes.pem \\
  --tls-private-key-file=/var/lib/kubernetes/kubernetes-key.pem \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

```
### Configure the Kubernetes Controller Manager

Move the `kube-controller-manager` kubeconfig into place:

```
sudo mv kube-controller-manager.kubeconfig /var/lib/kubernetes/
```

Create the `kube-controller-manager.service` systemd unit file:

```
cat <<EOF | sudo tee /etc/systemd/system/kube-controller-manager.service
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-controller-manager \\
  --bind-address=0.0.0.0 \\
  --cluster-cidr=10.200.0.0/16 \\
  --cluster-name=kubernetes \\
  --cluster-signing-cert-file=/var/lib/kubernetes/ca.pem \\
  --cluster-signing-key-file=/var/lib/kubernetes/ca-key.pem \\
  --kubeconfig=/var/lib/kubernetes/kube-controller-manager.kubeconfig \\
  --leader-elect=true \\
  --root-ca-file=/var/lib/kubernetes/ca.pem \\
  --service-account-private-key-file=/var/lib/kubernetes/service-account-key.pem \\
  --service-cluster-ip-range=10.32.0.0/24 \\
  --use-service-account-credentials=true \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```
### Configure the Kubernetes Scheduler

Move the `kube-scheduler` kubeconfig into place:

```
sudo mv kube-scheduler.kubeconfig /var/lib/kubernetes/
```

Create the `kube-scheduler.yaml` configuration file:

```
cat <<EOF | sudo tee /etc/kubernetes/config/kube-scheduler.yaml
apiVersion: kubescheduler.config.k8s.io/v1beta1
kind: KubeSchedulerConfiguration
clientConnection:
  kubeconfig: "/var/lib/kubernetes/kube-scheduler.kubeconfig"
leaderElection:
  leaderElect: true
EOF
```

Create the `kube-scheduler.service` systemd unit file:

```
cat <<EOF | sudo tee /etc/systemd/system/kube-scheduler.service
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-scheduler \\
  --config=/etc/kubernetes/config/kube-scheduler.yaml \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```
### Start the Controller Services

```
# systemctl daemon-reload
# systemctl enable kube-apiserver kube-controller-manager kube-scheduler
# systemctl start kube-apiserver kube-controller-manager kube-scheduler
```
> Allow up to 10 seconds for the Kubernetes API Server to fully initialize.
### Enable HTTP Health Checks
- Why Do We Need to Enable HTTP Health Checks? 
    - In Kelsey Hightower’s original Kubernetes the Hard Way guide, he uses a Good Cloud Platform (GCP) load balancer. The load balancer needs to  be able to perform health checks against the Kubernetes API to measure the health status of API nodes. The GCP load balancer cannot easily perform health checks over HTTPS, so the guide instructs us to set up a proxy server to allow these health checks to be performed over HTTP. Since we are using Nginx as our load balancer, we don’t actually need to do this, but it will be good practice for us. This exercise will help you understand the methods used in the original guide.
    - Part of Kelsey Hightower's original Kubernetes the Hard Way guide involves setting up an nginx proxy on each controller to provide access    to           
    - the Kubernetes API /healthz endpoint over http  This lesson explains the reasoning behind the inclusion of that step and guides you through  
    
    - the process of implementing the http /healthz proxy. You can set up a basic nginx proxy for the healthz endpoint by first installing nginx" 
> The `/healthz` API server endpoint does not require authentication by default.

Install a basic web server to handle HTTP health checks:
```
# yum install epel-release  nginx -y
```
Create an nginx configuration for the health check proxy: 
```
cat > /etc/nginx/conf.d/kubernetes.default.svc.cluster.local.conf <<EOF
server {
  listen      80;
  server_name kubernetes.default.svc.cluster.local;

  location /healthz {
     proxy_pass                    https://127.0.0.1:6443/healthz;
     proxy_ssl_trusted_certificate /var/lib/kubernetes/ca.pem;
  }
}
EOF
```
Started and enabled nginx 
```
# systemctl enable --now nginx
```
### Verification

```
kubectl cluster-info --kubeconfig admin.kubeconfig
```

```
Kubernetes control plane is running at https://127.0.0.1:6443
```

Test the nginx HTTP health check proxy:

```
curl -H "Host: kubernetes.default.svc.cluster.local" -i http://127.0.0.1/healthz
```

```
HTTP/1.1 200 OK
Server: nginx/1.18.0 (Ubuntu)
Date: Sun, 02 May 2021 04:19:29 GMT
Content-Type: text/plain; charset=utf-8
Content-Length: 2
Connection: keep-alive
Cache-Control: no-cache, private
X-Content-Type-Options: nosniff
X-Kubernetes-Pf-Flowschema-Uid: c43f32eb-e038-457f-9474-571d43e5c325
X-Kubernetes-Pf-Prioritylevel-Uid: 8ba5908f-5569-4330-80fd-c643e7512366

ok
```

> Remember to run the above commands on each controller node: `kubecon01`, `kubecon02`

## RBAC for Kubelet Authorization
One of the necessary steps in setting up a new Kubernetes cluster from scratch is to assign permissions that allow the Kubernetes API to access various functionality within the worker kubelets. This lesson guides you through the process of creating a ClusterRole and binding it to the kubernetes user so that those permissions will be in place. After completing this lesson, your cluster will have the necessary role-based access control configuration to allow the cluster's API to access kubelet functionality such as logs and metrics. You can configure RBAC for kubelet authorization with these commands. Note that these commands only need to be run on one control node. Create a role with the necessary permissions

In this section you will configure RBAC permissions to allow the Kubernetes API Server to access the Kubelet API on each worker node. Access to the Kubelet API is required for retrieving metrics, logs, and executing commands in pods.

> This tutorial sets the Kubelet `--authorization-mode` flag to `Webhook`. Webhook mode uses the [SubjectAccessReview](https://kubernetes.io/docs/admin/authorization/#checking-api-access) API to determine authorization.

The commands in this section will effect the entire cluster and only need to be run once from one of the controller nodes.

Create the `system:kube-apiserver-to-kubelet` [ClusterRole](https://kubernetes.io/docs/admin/authorization/rbac/#role-and-clusterrole) with permissions to access the Kubelet API and perform most common tasks associated with managing pods:

```
cat <<EOF | kubectl apply --kubeconfig admin.kubeconfig -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:kube-apiserver-to-kubelet
rules:
  - apiGroups:
      - ""
    resources:
      - nodes/proxy
      - nodes/stats
      - nodes/log
      - nodes/spec
      - nodes/metrics
    verbs:
      - "*"
EOF
```

The Kubernetes API Server authenticates to the Kubelet as the `kubernetes` user using the client certificate as defined by the `--kubelet-client-certificate` flag.

Bind the `system:kube-apiserver-to-kubelet` ClusterRole to the `kubernetes` user:

```
cat <<EOF | kubectl apply --kubeconfig admin.kubeconfig -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: system:kube-apiserver
  namespace: ""
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:kube-apiserver-to-kubelet
subjects:
  - apiGroup: rbac.authorization.k8s.io
    kind: User
    name: kubernetes
EOF
```


### Setting up a Kube API Frontend Load Balancer
In order to achieve redundancy for your Kubernetes cluster, you will need to load balance usage of the Kubernetes API across multiple control nodes. In this lesson, you will learn how to create a simple nginx server to perform this balancing. After completing this lesson, you will be able to interact with both control nodes of your kubernetes cluster using the nginx load balancer. Here are the commands you can use to set up the nginx load balancer. Run these on the server that you have designated as your load balancer server:  
```
# ssh root@192.168.0.3
```
Note will use stream module for nginx for easy wiil create a docker image that contain the all moudle config first lets install docker 

```
# yum install -y yum-utils
# yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo
# yum-config-manager --enable docker-ce-nightly
# yum install docker-ce docker-ce-cli containerd.io -y  
# systemctl enable --now docker
```
### Config the Loadbalancer 
Now will create the dir for image to configure all image nassry 

```
# mkdir nginx && cd nginx 
```
Set up some environment variables for the lead balancer config file: 

```
# CONTROLLER0_IP=192.168.0.1
# CONTROLLER1_IP=192.168.0.2
```
Create the load balancer nginx config file

```
# cat << EOF | sudo tee k8s.conf
   stream {
    upstream kubernetes {
        least_conn;
        server $CONTROLLER0_IP:6443;
        server $CONTROLLER1_IP:6443;
     }
    server {
        listen 6443;
        listen 443;
        proxy_pass kubernetes;
    }
}
EOF
```
Lets create a nginx config 
```
# cat << EOF |  tee nginx.conf 
user www-data;
worker_processes auto;
pid /run/nginx.pid;

events {
        worker_connections 768;
        # multi_accept on;
}

http {

        ##
        # Basic Settings
        ##

        sendfile on;
        tcp_nopush on;
        tcp_nodelay on;
        keepalive_timeout 65;
        types_hash_max_size 2048;
        # server_tokens off;

        # server_names_hash_bucket_size 64;
        # server_name_in_redirect off;

        include /etc/nginx/mime.types;
        default_type application/octet-stream;

        ##
        # SSL Settings
        ##

        ssl_protocols TLSv1 TLSv1.1 TLSv1.2; # Dropping SSLv3, ref: POODLE
        ssl_prefer_server_ciphers on;

        ##
        # Logging Settings
        ##

        access_log /var/log/nginx/access.log;
        error_log /var/log/nginx/error.log;

        ##
        # Gzip Settings
        ##

        gzip on;
        gzip_disable "msie6";

        # gzip_vary on;
        # gzip_proxied any;
        # gzip_comp_level 6;
        # gzip_buffers 16 8k;
        # gzip_http_version 1.1;
        # gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;

        ##
        # Virtual Host Configs
        ##

        include /etc/nginx/conf.d/*.conf;
        include /etc/nginx/sites-enabled/*;
}


#mail {
#       # See sample authentication script at:
#       # http://wiki.nginx.org/ImapAuthenticateWithApachePhpScript
#
#       # auth_http localhost/auth.php;
#       # pop3_capabilities "TOP" "USER";
#       # imap_capabilities "IMAP4rev1" "UIDPLUS";
#
#       server {
#               listen     localhost:110;
#               protocol   pop3;
#               proxy      on;
#       }
#
#       server {
#               listen     localhost:143;
#               protocol   imap;
#               proxy      on;
#       }
#}
include /etc/nginx/tcpconf.d/*;
EOF
```
Lets create a Dockerfile 

```
# cat << EOF |  tee Dockerfile
FROM ubuntu:16.04
RUN apt-get update -y && apt-get upgrade -y && apt-get install -y nginx && mkdir -p  /etc/nginx/tcpconf.d
RUN rm -rf /etc/nginx/nginx.conf
ADD nginx.conf /etc/nginx/
ADD k8s.conf /etc/nginx/tcpconf.d/
CMD ["nginx", "-g", "daemon off;"]
EOF
```
Lets build and run docker 

```
# docker build -t nginx .
# docker run -d --network host --name nginx --restart unless-stopped nginx
```
Make a HTTP request for the Kubernetes version info:

```
curl --cacert ca.pem https://${KUBERNETES_PUBLIC_ADDRESS}:6443/version
```

> output

```
{
  "major": "1",
  "minor": "21",
  "gitVersion": "v1.21.0",
  "gitCommit": "cb303e613a121a29364f75cc67d3d580833a7479",
  "gitTreeState": "clean",
  "buildDate": "2021-04-08T16:25:06Z",
  "goVersion": "go1.16.1",
  "compiler": "gc",
  "platform": "linux/amd64"
}
```
Next: [Bootstrapping the Kubernetes Worker Nodes](08-bootstrapping-kubernetes-workers.md)