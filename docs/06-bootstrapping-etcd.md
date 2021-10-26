# Bootstrapping the etcd Cluster

Kubernetes components are stateless and store cluster state in [etcd](https://github.com/etcd-io/etcd). In this lab you will bootstrap a three node etcd cluster and configure it for high availability and secure remote access.

## What Is etcd?
“etcd is a distributed key value store that provides a reliable way to store data across a cluster of machines.” [etcd](https://coreos.com/etcd/) etcd provides a way to store data across a distributed cluster of machines and make sure the data is synchronized across all machines. You can find more information, as well as the etcd source code, in the etcd GitHub repository [etcd](https://github.com/etcd-io/etcd)

## How Is etcd Used in Kubernetes? 
Kubernetes uses etcd to store all of its internal data about cluster state. This data needs to be stored, but it also needs to be reliably synchronized across all controller nodes in the cluster. etcd fulfills that purpose. We will need to install etcd on each of our Kubernetes controller nodes and create an etcd cluster that includes all of those controller nodes. You can find more information on managing an etcd cluster for Kubernetes here [k8setcd](https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/)

## Creating the etcd Cluster
Before you can stand up controllers for a Kubernetes cluster, you must first build an etcd cluster across your Kubernetes control nodes. This lesson provides a demonstration of how to set up an etcd cluster in preparation for bootstrapping Kubernetes. After completing this lesson, you should have a working etcd cluster that consists of your Kubernetes control nodes. Here are the commands used in the demo (note that these have to be run on both controller servers, with a few differences between them): 
### Download and Install the etcd Binaries

Download the official etcd release binaries from the [etcd](https://github.com/etcd-io/etcd) GitHub project:

```
wget "https://github.com/etcd-io/etcd/releases/download/v3.4.15/etcd-v3.4.15-linux-amd64.tar.gz"
```
Extract and install the `etcd` server and the `etcdctl` command line utility:

```
# tar -xvf etcd-v3.4.15-linux-amd64.tar.gz
# mv etcd-v3.4.15-linux-amd64/etcd* /usr/local/bin/
```
### Configure the etcd Server
```
# mkdir -p /etc/etcd /var/lib/etcd
# chmod 700 /var/lib/etcd
# cp ca.pem kubernetes-key.pem kubernetes.pem /etc/etcd/
```
Set up the following environment variables. 
```
# ETCD_NAME=$(hostname -s)
# INTERNAL_IP=$(/sbin/ip -o -4 addr list eth0 | awk '{print $4}' | cut -d/ -f1)
```
Set up the following environment variables. Be sure you replace all of the with their corresponding real values: 
> you can see the lab digram  in your case you only to change the ip for varaible 
![digram screenshot](images/k8s_digram)
```
# CONTROLLER0_IP=192.168.0.1
# CONTROLLER0_host=kubecon01
# CONTROLLER1_IP=192.168.0.2
# CONTROLLER1_host=kubecon02
```
Create the `etcd.service` systemd unit file:

```
cat <<EOF | sudo tee /etc/systemd/system/etcd.service
[Unit]
Description=etcd
Documentation=https://github.com/coreos

[Service]
Type=notify
ExecStart=/usr/local/bin/etcd \\
  --name ${ETCD_NAME} \\
  --cert-file=/etc/etcd/kubernetes.pem \\
  --key-file=/etc/etcd/kubernetes-key.pem \\
  --peer-cert-file=/etc/etcd/kubernetes.pem \\
  --peer-key-file=/etc/etcd/kubernetes-key.pem \\
  --trusted-ca-file=/etc/etcd/ca.pem \\
  --peer-trusted-ca-file=/etc/etcd/ca.pem \\
  --peer-client-cert-auth \\
  --client-cert-auth \\
  --initial-advertise-peer-urls https://${INTERNAL_IP}:2380 \\
  --listen-peer-urls https://${INTERNAL_IP}:2380 \\
  --listen-client-urls https://${INTERNAL_IP}:2379,https://127.0.0.1:2379 \\
  --advertise-client-urls https://${INTERNAL_IP}:2379 \\
  --initial-cluster-token etcd-cluster-0 \\
  --initial-cluster ${CONTROLLER0_host}=https://${CONTROLLER0_IP}:2380,${CONTROLLER1_host}=https://${CONTROLLER1_IP}:2380 \\
  --initial-cluster-state new \\
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```
### Start the etcd Server

```
# systemctl daemon-reload
# systemctl enable etcd
# systemctl start etcd
```
> Remember to run the above commands on each controller node: `kubecon01`, `kubecon02` .

## Verification

List the etcd cluster members:

```
sudo ETCDCTL_API=3 etcdctl member list \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/etcd/ca.pem \
  --cert=/etc/etcd/kubernetes.pem \
  --key=/etc/etcd/kubernetes-key.pem
```
> output
19e6cf768d9d542e, started, kubecon02, https://192.168.0.2:2380, https://192.168.0.2:2379, false
508e54ff346cdb88, started, kubecon01, https://192.168.0.1:2380, https://192.168.0.1:2379, false

Next: [Bootstrapping the Kubernetes Control Plane](07-bootstrapping-kubernetes-controllers.md)