## Running k3s using multipass

### Create master node and worker nodes
1. Install multipass using snap
```
$ sudo snap install multipass
```
2. Create master node and install k3s
```
$ multipass launch --name master -m 4G -d 20G
$ multipass exec master -- /bin/bash -c "curl -sfL https://get.k3s.io | K3S_KUBECONFIG_MODE='644' sh -"
```
3. Get the master node IP address and the k3s token
```
$ multipass info master | grep IPv4
IPv4:           10.165.182.250
$ TOKEN="$(multipass exec master -- /bin/bash -c 'sudo cat /var/lib/rancher/k3s/server/node-token')"
```
4. Create the worker nodes
```
$ multipass launch --name worker1 -m 4G -d 20G
$ multipass launch --name worker2 -m 4G -d 20G
```
5. Install k3s on the worker, using the master's IP address and token
```
$ multipass exec worker1 -- /bin/bash -c "curl -sfL https://get.k3s.io | K3S_TOKEN=${TOKEN} K3S_URL='https://10.165.182.250:6443' sh -"
$ multipass exec worker2 -- /bin/bash -c "curl -sfL https://get.k3s.io | K3S_TOKEN=${TOKEN} K3S_URL='https://10.165.182.250:6443' sh -"
```
6. Go to master, and set the bash_profile and test the traefik routing
```
$ multipass shell master
ubuntu@master:~$ vi ~/.bash_profile
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
ubuntu@master:~$ vi index.html
<html>
<head>
  <title>Hello World!</title>
</head>
<body>Hello World!</body>
</html>
ubuntu@master:~$ kubectl create configmap hello-world --from-file index.html
ubuntu@master:~$ kubectl apply -f hello_route.yaml 
ubuntu@master:~$ kubectl get po --all-namespaces
NAMESPACE     NAME                                      READY   STATUS      RESTARTS      AGE
default       hello-world-nginx-6dfb89b8bf-hp9sw        1/1     Running     2 (29m ago)   23h
kube-system   coredns-7b98449c4-cvc8q                   1/1     Running     2 (29m ago)   25h
kube-system   helm-install-traefik-crd-mpfm8            0/1     Completed   0             29m
kube-system   helm-install-traefik-zlcdx                0/1     Completed   1             25h
kube-system   local-path-provisioner-595dcfc56f-kh4p9   1/1     Running     2 (29m ago)   25h
kube-system   metrics-server-cdcc87586-n25kt            1/1     Running     2 (29m ago)   25h
kube-system   svclb-traefik-32f6769d-6f4mf              2/2     Running     2 (22m ago)   25h
kube-system   svclb-traefik-32f6769d-kth8t              2/2     Running     4 (29m ago)   25h
kube-system   svclb-traefik-32f6769d-rzskb              2/2     Running     0             25h
kube-system   traefik-d7c9c5778-22s9h                   1/1     Running     2 (29m ago)   25h
ubuntu@master:~$ kubectl get svc -n kube-system
NAME             TYPE           CLUSTER-IP    EXTERNAL-IP                    PORT(S)                      AGE
kube-dns         ClusterIP      10.43.0.10    <none>                         53/UDP,53/TCP,9153/TCP       25h
metrics-server   ClusterIP      10.43.83.75   <none>                         443/TCP                      25h
traefik          LoadBalancer   10.43.16.24   10.165.182.250,10.165.182.47   80:31935/TCP,443:32387/TCP   25h
ubuntu@master:~$ 
ubuntu@master:~$ curl http://10.165.182.47/hello
<html>
<head>
  <title>Hello World!</title>
</head>
<body>Hello World!</body>
</html>
ubuntu@master:~$ 

```

### Notes
To install helm in the master node:
```
sudo snap install helm --classic
```

Useful multipass commands
```
multipass list
multipass stop master
multipass start master
```

Useful k3 info
- Location of manifests directory: /var/lib/rancher/k3s/server/manifests
