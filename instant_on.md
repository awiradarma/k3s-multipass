## Experiment with InstantOn on k3s

Exporting an instant-on container image to tar ball
```
andre@cyber:~/workspace/graalvm_comparison/Simple$ sudo podman save localhost/sb_after:latest -o sb_after.tar
Copying blob f72362c98463 done   | 
Copying blob 5bf71d626863 done   | 
Copying blob c9d8f6764375 done   | 
Copying blob 414aa6a23471 done   | 
Copying blob e49971ebea06 done   | 
Copying blob c6c2eaf82be9 done   | 
Copying blob 4826bfe16f25 done   | 
Copying blob 51cb3aabc02f done   | 
Copying blob ef1e14cee31b done   | 
Copying blob 01db667bedeb done   | 
Copying blob 4614a8c00c49 done   | 
Copying blob a0aecc0e3ced done   | 
Copying blob 237b6ae9070d done   | 
Copying blob 8bee85d39414 done   | 
Copying blob 390db0b61b2b done   | 
Copying blob d455dd84253e done   | 
Copying blob 9a7bc69028aa done   | 
Copying blob 32d385bc1f29 done   | 
Copying blob 2c46d2f69a3e done   | 
Copying blob 54ee051a2449 done   | 
Copying blob e7c429ef0cc6 done   | 
Copying blob 99e333962e2c done   | 
Copying config b98acdeb88 done   | 
Writing manifest to image destination
andre@cyber:~/workspace/graalvm_comparison/Simple$ ls -ltr
total 565656
-rw-rw-r--  1 andre andre      7061 Nov  6 19:25 mvnw.cmd
-rwxrwxr-x  1 andre andre     10665 Nov  6 19:25 mvnw
drwxrwxr-x  4 andre andre      4096 Nov  6 19:25 src
-rw-rw-r--  1 andre andre      1396 Nov  6 19:51 Dockerfile
-rw-rw-r--  1 andre andre      3017 Nov  6 19:56 output.txt
-rw-rw-r--  1 andre andre      1876 Nov  6 20:01 pom.xml
drwxrwxr-x 11 andre andre      4096 Nov  6 20:03 target
-rw-rw-r--  1 andre andre      7237 Nov  6 20:15 output_local.txt
-rw-rw-r--  1 andre andre      2703 Nov  7 20:22 output_instanton.txt
-rw-rw-r--  1 andre andre      1428 Nov  7 20:22 Dockerfile.instanton
-rw-rw-r--  1 andre andre     13560 Nov  7 20:25 output_instanton_afterAppStart.txt
-rw-r--r--  1 root  root  579154432 Nov  7 20:47 sb_after.tar
```

Transferring the tar ball to the multipass VM
```
$ multipass transfer sb_after.tar master:.
```

On the multipass VM, load the tar ball
```
ubuntu@master:~$ ls -ltr
total 565600
drwx------ 3 ubuntu ubuntu      4096 Nov  2 20:22 snap
-rw-rw-r-- 1 ubuntu ubuntu        86 Nov  2 20:50 index.html
-rw-rw-r-- 1 ubuntu ubuntu      1067 Nov  2 21:21 hello_route.yaml
-rw-r--r-- 1 ubuntu ubuntu 579154432 Nov  7 20:47 sb_after.tar

ubuntu@master:~$ sudo k3s crictl images
IMAGE                                        TAG                    IMAGE ID            SIZE
docker.io/library/nginx                      latest                 3b25b682ea82b       73MB
docker.io/rancher/klipper-helm               v0.9.3-build20241008   4e0aed78b287d       70.5MB
docker.io/rancher/klipper-lb                 v0.4.9                 11a5d8a9f31aa       4.99MB
docker.io/rancher/local-path-provisioner     v0.0.30                b580d47bc23dd       18.6MB
docker.io/rancher/mirrored-coredns-coredns   1.11.3                 c69fa2e9cbf5f       18.6MB
docker.io/rancher/mirrored-library-traefik   2.11.10                1741c0b1ff49b       47MB
docker.io/rancher/mirrored-metrics-server    v0.7.2                 48d9cfaaf3904       19.5MB
docker.io/rancher/mirrored-pause             3.6                    6270bb605e12e       301kB
ubuntu@master:~$ sudo k3s ctr -n=k8s.io images import sb_after.tar
unpacking localhost/sb_after:latest (sha256:04872eae0c16a2d1f036a8bff2efc719a0cbfe13dad025048e75a317cfc34618)...done
ubuntu@master:~$ sudo k3s crictl images
IMAGE                                        TAG                    IMAGE ID            SIZE
docker.io/library/nginx                      latest                 3b25b682ea82b       73MB
docker.io/rancher/klipper-helm               v0.9.3-build20241008   4e0aed78b287d       70.5MB
docker.io/rancher/klipper-lb                 v0.4.9                 11a5d8a9f31aa       4.99MB
docker.io/rancher/local-path-provisioner     v0.0.30                b580d47bc23dd       18.6MB
docker.io/rancher/mirrored-coredns-coredns   1.11.3                 c69fa2e9cbf5f       18.6MB
docker.io/rancher/mirrored-library-traefik   2.11.10                1741c0b1ff49b       47MB
docker.io/rancher/mirrored-metrics-server    v0.7.2                 48d9cfaaf3904       19.5MB
docker.io/rancher/mirrored-pause             3.6                    6270bb605e12e       301kB
localhost/sb_after                           latest                 b98acdeb88614       579MB
ubuntu@master:~$ 

```

## Create a deployment 

The yaml file
```
ubuntu@master:~$ cat instanton.yaml 
apiVersion: apps/v1
kind: Deployment
metadata:
  name: open-liberty-instanton
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: open-liberty-instanton
  template:
    metadata:
      labels:
        app.kubernetes.io/name: open-liberty-instanton
    spec:
      containers:
      - image: localhost/sb_after
        imagePullPolicy: IfNotPresent
        name: app
        ports:
        - containerPort: 9080
          name: 9080-tcp
          protocol: TCP
        resources:
          limits:
            cpu: 1
            memory: 512Mi
          requests:
            cpu: 500m
            memory: 256Mi
        securityContext:
          runAsNonRoot: true
          privileged: false
          capabilities:
            add:
            - CHECKPOINT_RESTORE
            - SETPCAP
            drop:
            - ALL

```

Deploy the YAML and check the log file
```
ubuntu@master:~$ kubectl get po
NAME                                 READY   STATUS    RESTARTS      AGE
hello-world-nginx-6dfb89b8bf-hp9sw   1/1     Running   3 (22m ago)   5d
ubuntu@master:~$ 
ubuntu@master:~$ 
ubuntu@master:~$ kubectl apply -f instanton.yaml
deployment.apps/open-liberty-instanton created
ubuntu@master:~$ 
ubuntu@master:~$ 
ubuntu@master:~$ kubectl get po
NAME                                      READY   STATUS    RESTARTS      AGE
hello-world-nginx-6dfb89b8bf-hp9sw        1/1     Running   3 (22m ago)   5d
open-liberty-instanton-57847f5fcc-whv9f   1/1     Running   0             3s
ubuntu@master:~$ kubectl logs open-liberty-instanton-57847f5fcc-whv9f

[AUDIT   ] Launching defaultServer (Open Liberty 24.0.0.11/wlp-1.0.95.cl241120241021-1102) on Eclipse OpenJ9 VM, version 21.0.4+7-LTS (en_US)
[AUDIT   ] CWWKT0016I: Web application available (default_host): http://open-liberty-instanton-57847f5fcc-whv9f:9080/
[AUDIT   ] CWWKC0452I: The Liberty server process resumed operation from a checkpoint in 0.076 seconds.
[AUDIT   ] CWWKZ0001I: Application thin-Simple-0.0.1-SNAPSHOT started in 0.081 seconds.
[AUDIT   ] CWWKF0012I: The server installed the following features: [expressionLanguage-5.0, pages-3.1, servlet-6.0, springBoot-3.0, ssl-1.0, transportSecurity-1.0, websocket-2.1].
[AUDIT   ] CWWKF0011I: The defaultServer server is ready to run a smarter planet. The defaultServer server started in 0.085 seconds.
ubuntu@master:~$ 

```