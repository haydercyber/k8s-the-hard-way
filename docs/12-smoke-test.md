## Smoke Test

In this lab you will complete a series of tasks to ensure your Kubernetes cluster is functioning correctly.
Now we want to run some basic smoke tests to make sure everything in our cluster is working correctly. We will test the following features: 
- Data encryption
- Deployments
- Port forwarding
- Logs
- Exec
- Services
- Untrusted workloads


## Data Encryption

In this section you will verify the ability to [encrypt secret data at rest](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/#verifying-that-data-is-encrypted).

Create a generic secret:
- Goal: 
    - Verify that we can encrypt secret data at rest. 
- Strategy:
    - Create a generic secret in the cluster. 
    - Dump the raw data from etcd and verify that it is encrypted
we set up a data encryption config to allow Kubernetes to encrypt sensitive data. In this lesson, we will smoke test that functionality by creating some secret data and verifying that it is stored in an encrypted format in etcd. After completing this lesson, you will have verified that your cluster can successfully encrypt sensitive data
```
kubectl create secret generic kubernetes-the-hard-way \
  --from-literal="mykey=mydata"
```

Print a hexdump of the `kubernetes-the-hard-way` secret stored in etcd:
Log in to one of your controller servers, and get the raw data for the test secret from etcd
```
#  ETCDCTL_API=3 etcdctl get   \
   --endpoints=https://127.0.0.1:2379 \
   --cacert=/etc/etcd/ca.pem \
   --cert=/etc/etcd/kubernetes.pem \
   --key=/etc/etcd/kubernetes-key.pem\
    /registry/secrets/default/kubernetes-the-hard-way | hexdump -C
```
> output

```
00000000  2f 72 65 67 69 73 74 72  79 2f 73 65 63 72 65 74  |/registry/secret|
00000010  73 2f 64 65 66 61 75 6c  74 2f 6b 75 62 65 72 6e  |s/default/kubern|
00000020  65 74 65 73 2d 74 68 65  2d 68 61 72 64 2d 77 61  |etes-the-hard-wa|
00000030  79 0a 6b 38 73 3a 65 6e  63 3a 61 65 73 63 62 63  |y.k8s:enc:aescbc|
00000040  3a 76 31 3a 6b 65 79 31  3a ea 5f 64 1f 22 63 ac  |:v1:key1:._d."c.|
00000050  e5 a0 2d 7f 1e cd e3 03  64 a0 8e 7f cf 58 db 50  |..-.....d....X.P|
00000060  d7 d0 12 a1 31 2e 72 53  e3 51 de 31 53 96 d7 3f  |....1.rS.Q.1S..?|
00000070  71 f5 e3 3f 07 bc 33 56  55 ed 9c 67 6a 91 77 18  |q..?..3VU..gj.w.|
00000080  52 bb ad 61 64 76 43 df  00 b5 aa 7e 8e cb 16 e9  |R..advC....~....|
00000090  9b 5a 21 04 49 37 63 a5  6c df 09 b7 2b 5c 96 69  |.Z!.I7c.l...+\.i|
000000a0  02 03 42 02 93 7d 42 57  c9 8d 28 2d 1c 9d dd 2b  |..B..}BW..(-...+|
000000b0  a3 69 fa ca c8 8f a0 0e  66 c8 5b 5a 40 29 80 0d  |.i......f.[Z@)..|
000000c0  06 c3 56 87 27 ba d2 19  a6 b0 e6 b5 70 b3 18 02  |..V.'.......p...|
000000d0  69 ed ae b1 4d 03 be 92  08 9e 20 62 41 cd e6 a4  |i...M..... bA...|
000000e0  8c e0 fd b0 5f 44 11 a1  e0 99 a4 61 71 b2 c2 98  |...._D.....aq...|
000000f0  b1 f3 bf 48 a5 26 11 8c  9e 4e 12 7a 81 f4 20 11  |...H.&...N.z.. .|
00000100  05 0d db 62 82 53 2c d9  71 0d 9f af d7 e2 b6 94  |...b.S,.q.......|
00000110  4c 67 98 2e 66 21 77 5e  ea 4d f5 23 6c d4 4b 56  |Lg..f!w^.M.#l.KV|
00000120  58 a7 f1 3b 23 8d 5b 45  14 2c 05 3a a9 90 95 a4  |X..;#.[E.,.:....|
00000130  9a 5f 06 cc 42 65 b3 31  d8 9c 78 a9 f1 da a2 81  |._..Be.1..x.....|
00000140  5a a6 f6 d8 7c 2e 8c 13  f0 30 b1 25 ab 6e bb 2f  |Z...|....0.%.n./|
00000150  cd 7f fd 44 98 64 97 9b  31 0a                    |...D.d..1.|
0000015a
```
The etcd key should be prefixed with `k8s:enc:aescbc:v1:key1`, which indicates the `aescbc` provider was used to encrypt the data with the `key1` encryption key.

### Deployments
- Goal:
    - Verify that we can create a deployment and that it can successfully create pods. 
- Strategy:
    - Create a simple deployment. 
    - Verify that the deployment successfully creates a pod
Deployments are one of the powerful orchestration tools offered by Kubernetes. In this lesson, we will make sure that deployments are working in our cluster. We will verify that we can create a deployment, and that the deployment is able to successfully stand up a new pod and container.
In this section you will verify the ability to create and manage [Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/).

Create a deployment for the [nginx](https://nginx.org/en/) web server:

```
kubectl create deployment nginx --image=nginx
```

List the pod created by the `nginx` deployment:

```
kubectl get pods -l app=nginx
```

> output
```
nginx-6799fc88d8-vtz4c   1/1     Running   0          21s
```
### Port Forwarding
- Goal:
    - Verify that we can use port forwarding to access pods remotely
- Strategy:
    - Use kubectl port-forward to set up port forwarding for an Nginx pod
    - Access the pod remotely with curl.
In this section you will verify the ability to access applications remotely using [port forwarding](https://kubernetes.io/docs/tasks/access-application-cluster/port-forward-access-application-cluster/).

Retrieve the full name of the `nginx` pod:

```
POD_NAME=$(kubectl get pods -l app=nginx -o jsonpath="{.items[0].metadata.name}")
```

Forward port `8080` on your local machine to port `80` of the `nginx` pod:

```
kubectl port-forward $POD_NAME 8080:80
```

> output
```
Forwarding from 127.0.0.1:8080 -> 80
Forwarding from [::1]:8080 -> 80
```
In a new terminal make an HTTP request using the forwarding address:

```
curl --head http://127.0.0.1:8080
```

> output

```
HTTP/1.1 200 OK
Server: nginx/1.19.10
Date: Sun, 02 May 2021 05:29:25 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Tue, 13 Apr 2021 15:13:59 GMT
Connection: keep-alive
ETag: "6075b537-264"
Accept-Ranges: bytes
```

Switch back to the previous terminal and stop the port forwarding to the `nginx` pod:

```
Forwarding from 127.0.0.1:8080 -> 80
Forwarding from [::1]:8080 -> 80
Handling connection for 8080
^C
```

### Logs
- Goal:
    - Verify that we can get container logs with kubectl logs.
- Strategy: 
    - Get the logs from the Nginx pod container.
When managing a cluster, it is often necessary to access container logs to check their health and diagnose issues. Kubernetes offers access to container logs via the kubectl logs command. In this lesson, In this section you will verify the ability to [retrieve container logs](https://kubernetes.io/docs/concepts/cluster-administration/logging/).

Print the `nginx` pod logs:

```
kubectl logs $POD_NAME
```

> output

```
2021/10/26 18:07:29 [notice] 1#1: start worker processes
2021/10/26 18:07:29 [notice] 1#1: start worker process 30
2021/10/26 18:07:29 [notice] 1#1: start worker process 31
127.0.0.1 - - [26/Oct/2021:18:20:44 +0000] "HEAD / HTTP/1.1" 200 0 "-" "curl/7.29.0" "-"
```
### Exec 

- Goal: 
    - Verify that we can run commands in a container with kubectl exec
- Strategy: 
    - Use kubectl exec to run a command in the Nginx pod container.
The kubectl exec command is a powerful management tool that allows us to run commands inside of Kubernetes-managed containers. In order to verify that our cluster is set up correctly, we need to make sure that kubectl exec is working. 
In this section you will verify the ability to [execute commands in a container](https://kubernetes.io/docs/tasks/debug-application-cluster/get-shell-running-container/#running-individual-commands-in-a-container).

Print the nginx version by executing the `nginx -v` command in the `nginx` container:

```
kubectl exec -ti $POD_NAME -- nginx -v
```

> output

```
nginx version: nginx/1.21.3
```

## Services
- Goal:
    - Verify that we can create and access services.
    - Verify that we can run an untrusted workload under gVisor (runsc) 
- Strategy: 
    - Create a NodePort service to expose the Nginx deployment.
    - Access the service remotely using the NodePort.
    - Run a pod as an untrusted workload.
    - Log in to the worker node that is running the pod and verify that its container is running using runsc.
In order to make sure that the cluster is set up correctly, we need to ensure that services can be created and accessed appropriately. In this lesson, we will smoke test our cluster's ability to create and access services by creating a simple testing service, and accessing it using a node port. If we can successfully create the service and use it to access our nginx pod, then we will know that our cluster is able to correctly handle services! 
In this section you will verify the ability to expose applications using a [Service](https://kubernetes.io/docs/concepts/services-networking/service/).

Expose the `nginx` deployment using a [NodePort](https://kubernetes.io/docs/concepts/services-networking/service/#type-nodeport) service:

```
kubectl expose deployment nginx --port 80 --type NodePort
```
Retrieve the node port assigned to the `nginx` service:

```
NODE_PORT=$(kubectl get svc nginx \
  --output=jsonpath='{range .spec.ports[0]}{.nodePort}')
```
Make an HTTP request using the external IP address and the `nginx` node port:

```
curl -I http://${EXTERNAL_IP}:${NODE_PORT}
```

> output

```
HTTP/1.1 200 OK
Server: nginx/1.19.10
Date: Sun, 02 May 2021 05:31:52 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Tue, 13 Apr 2021 15:13:59 GMT
Connection: keep-alive
ETag: "6075b537-264"
Accept-Ranges: bytes
```
Next: [Cleaning Up](13-cleanup.md)