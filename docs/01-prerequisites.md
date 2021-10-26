# Prerequisites

- its Need 6 centos vm 
    - A compatible Linux host. The Kubernetes project provides generic instructions 
    - 2 GB or more of RAM per machine (any less will leave little room for your apps).
    - 2 CPUs or more.
    - Full network connectivity between all machines in the cluster (public or private network is fine).
    -  Unique hostname, MAC address, and product_uuid for every node. See here for more details.
    - Swap disabled. You MUST disable swap in order for the kubelet to work properly
> you can see the lab digram  in your case you only to change the ip for your machine edit hostname and mapping to your machines ip then add to /etc/hosts
![digram screenshot](images/k8s_digram)

## editing host file 
note: the ip will change to your ip range 
```
# cat <<EOF>> /etc/hosts 
192.168.0.1 kubecon01.k8s.com
192.168.0.2 kubecon02.k8s.com
192.168.0.5 worknode01.k8s.com
192.168.0.6 worknode02.k8s.com
192.168.0.3 api_loadbalancer.k8s.com
EOF
```

## Install Some Package in machine will help you 
```
# yum install bash-completion vim telnet -y 
```
## make sure the firewalld servies is stop and disabled 
```
# systemctl disable --now firewalld
```
## make sure the Selinux is disabled 
```
# setenforce 0
# sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
```
Next: [Installing the Client Tools](02-client-tools.md)

