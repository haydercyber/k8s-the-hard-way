# Provisioning Pod Network Routes

In this lab you will use [calico ](https://docs.projectcalico.org/getting-started/kubernetes/)

> There are [other ways](https://kubernetes.io/docs/concepts/cluster-administration/networking/#how-to-achieve-this) to implement the Kubernetes networking model.

### The Kubernetes Networking Model
- What Problems Does the Networking Model Solve?
    - How will containers communicate with each other?
    - What if the containers are on different hosts (worker nodes)?
    - How will containers communicate with services?
    - How will containers be assigned unique IP addresses? What port(s) will be used?

### The Docker Model 
Docker allows containers to communicate with one another using a virtual network bridge configured on the host. Each host has its own virtual network serving all of the containers on that host. But what about containers on different hosts? We have to proxy traffic from the host to the containers, making sure no two containers use the same port on a host. The Kubernetes networking model was created in response to the Docker model. It was designed to improve on some of the limitations of the Docker model

### The Kubernetes Networking Model 
- One virtual network for the whole cluster.
- Each pod has a unique IP within the cluster.
- Each service has a unique IP that is in a different range than pod IPs.

### Cluster Network Architecture
Some Important CIDR ranges:
- Cluster CIDR
    - IP range used to assign IPs to pods in the cluster. In this course, we’ll be using a cluster CIDR of 10.200.0.0/16
- Service Cluster IP Range
    - IP range for services in the cluster. This should not overlap with the cluster CIDR range! In this course, our service cluster IP range is 10.32.0.0/24.
- Pod CIDR
    - IP range for pods on a specific worker node. This range should fall within the cluster CIDR but not overlap with the pod CIDR of any other worker node. In this course,    our networking plugin will automatically handle IP allocation to nodes, so we do not need to manually set a pod CIDR.
### Install Calico Networking  on  Kubernetes
We will be using calico Net to implement networking in our Kubernetes cluster.
We are now ready to set up networking in our Kubernetes cluster. This lesson guides you through the process of installing Weave Net in the cluster. It also shows you how to test your cluster network to make sure that everything is working as expected so far. After completing this lesson, you should have a functioning cluster network within your Kubernetes cluster. You can configure Weave Net like this: First, log in to both worker nodes and enable IP forwarding

```
# sysctl net.ipv4.conf.all.forwarding=1
# echo "net.ipv4.conf.all.forwarding=1" | sudo tee -a /etc/sysctl.conf
```
login in remote kubectl then install clico
```
# kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml 
```
> note it take between 10 to 15 min to be up 
Now calico Net is installed, but we need to test our network to make sure everything is working. First, make sure the calico Net pods are up and running: 

```
# kubectl get pods -n kube-system 
```
### Verification
Next, we want to test that pods can connect to each other and that they can connect to services. We will set up two Nginx pods and a service for those two pods. Then, we will create a busybox pod and use it to test connectivity to both Nginx pods and the service. First, create an Nginx deployment with 2 replicas:

```
#  cat << EOF |  tee nginx.yml 
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  selector:
     matchLabels:
      run: nginx
  replicas: 2
  template:
    metadata:
      labels:
        run: nginx
    spec:
      containers:
      - name: my-nginx
        image: nginx
        ports:
        - containerPort: 80
EOF
# kubectl apply -f nginx.yml
```
Next, create a service for that deployment so that we can test connectivity to services as well: 

```
# kubectl expose deployment/nginx 
```
Now let's start up another pod. We will use this pod to test our networking. We will test whether we can connect to the other pods and services from this pod.

```
# kubectl run busybox --image=radial/busyboxplus:curl --command -- sleep 3600
# POD_NAME=$(kubectl get pods -l run=busybox -o jsonpath="{.items[0].metadata.name}")
```
Now let's get the IP addresses of our two Nginx pods: 
```
# kubectl get ep nginx 
```
Now let's make sure the busybox pod can connect to the Nginx pods on both of those IP addresses
```
# kubectl exec $POD_NAME -- curl <first nginx pod IP address>
# kubectl exec $POD_NAME -- curl <second nginx pod IP address>
```
Both commands should return some HTML with the title "Welcome to Nginx!" This means that we can successfully connect to other pods. Now let's verify that we can connect to services. 
```
# kubectl get svc
```
Let's see if we can access the service from the busybox pod!

```
# kubectl exec $POD_NAME -- curl <nginx service IP address>
```
This should also return HTML with the title "Welcome to Nginx!" This means that we have successfully reached the Nginx service from inside a pod and that our networking configuration is working!
Now that we have networking set up in the cluster, we need to clean up the objects that were created in order to test the networking. These object could get in the way or become confusing in later lessons, so it is a good idea to remove them from the cluster before proceeding. After completing this lesson, your networking should still be in place, but the pods and services that were used to test it will be cleaned up. 

```
# kubectl get deploy 
# kubectl delete deployment nginx
# kubectl delete svc nginx
# kubectl delete pod busybox
```
Next: [Deploying the DNS Cluster Add-on](11-dns-addon.md)