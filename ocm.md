## Open Cluster Management
### Set up the clusters
- Use multipass to generate 3 VMs, 1 for each cluster
```
multipass launch --name ocmhub -m 4G -d 20G
multipass launch --name ocm1 -m 4G -d 20G
multipass launch --name ocm2 -m 4G -d 20G
```
- Install default k3s on each VM
```
multipass exec ocmhub -- /bin/bash -c "curl -sfL https://get.k3s.io | K3S_KUBECONFIG_MODE='644' sh -"
multipass exec ocm1 -- /bin/bash -c "curl -sfL https://get.k3s.io | K3S_KUBECONFIG_MODE='644' sh -"
multipass exec ocm2 -- /bin/bash -c "curl -sfL https://get.k3s.io | K3S_KUBECONFIG_MODE='644' sh -"
```
### Set up OCM Control Plane
Reference: https://open-cluster-management.io/docs/getting-started/installation/start-the-control-plane/
- Validate that kubectl and kustomize are installed
```
$ multipass shell ocmhub
$ kubectl version
Client Version: v1.30.6+k3s1
Kustomize Version: v5.0.4-0.20230601165947-6ce0bf390ce3
Server Version: v1.30.6+k3s1
$
```
- Install clusteradm CLI tool
```
curl -L https://raw.githubusercontent.com/open-cluster-management-io/clusteradm/main/install.sh | bash
```
- Bootstrap cluster manager
```
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
echo "export KUBECONFIG=/etc/rancher/k3s/k3s.yaml" >> ~/.bash_profile
clusteradm init --wait --context default
```

### Register the managed clusters
- Get your local IP address
```
awk '/32 host/ { print f } {f=$2}' /proc/net/fib_trie
```
- Install clusteradm
```
curl -L https://raw.githubusercontent.com/open-cluster-management-io/clusteradm/main/install.sh | bash
```
- Set the context
```
sudo cp /etc/rancher/k3s/k3s.yaml ocm1.yaml
sudo chown ubuntu:ubuntu ocm1.yaml
sed -i -e 's/127.0.0.1/<your local IP address>/g' ocm1.yaml
export KUBECONFIG=ocm1.yaml
```
- Bootstrap a klusterlet
```
clusteradm join --hub-token <token data from OCM Hub> --hub-apiserver https://<IP address of OCM Hub>:6443 --wait --cluster-name ocm1
```
- Back on the OCM Hub, accept the join request
```
clusteradm accept --clusters ocm1
```
- Validate 
```
$ kubectl get managedcluster 
NAME   HUB ACCEPTED   MANAGED CLUSTER URLS   JOINED   AVAILABLE   AGE
ocm1   true                                  True     True        4m46s
$ 
```
- Repeat steps on the second cluster (ocm2) - you might need to get a different token. Run the following on the OCM hub:
```
clusteradm get token
```
- Use clusteradm to check the clusters
```
ubuntu@ocmhub:~$ kubectl get managedcluster
NAME   HUB ACCEPTED   MANAGED CLUSTER URLS   JOINED   AVAILABLE   AGE
ocm1   true                                  True     True        96m
ocm2   true                                  True     True        21m
ubuntu@ocmhub:~$ clusteradm get clusters
<ManagedCluster> 
└── <ocm1> 
│   ├── <ClusterSet> default
│   ├── <KubernetesVersion> v1.30.6+k3s1
│   ├── <Capacity> 
│   │   ├── <Cpu> 1
│   │   ├── <Memory> 4004652Ki
│   ├── <Accepted> true
│   ├── <Available> True
└── <ocm2> 
    └── <ClusterSet> default
    └── <KubernetesVersion> v1.30.6+k3s1
    └── <Capacity> 
    │   ├── <Cpu> 1
    │   ├── <Memory> 4004652Ki
    └── <Accepted> true
    └── <Available> True
ubuntu@ocmhub:~$ clusteradm get clustersets
<ManagedClusterSet> 
└── <default> 
│   ├── <BoundNamespace> 
│   ├── <Status> 2 ManagedClusters selected
│   ├── <Clusters> [ocm1 ocm2]
└── <global> 
    └── <BoundNamespace> 
    └── <Status> 2 ManagedClusters selected
    └── <Clusters> [ocm1 ocm2]
ubuntu@ocmhub:~$ 
```
### Try deploying a workload in a managed cluster
- Create a manifest work yaml on the OCM hub
```
apiVersion: work.open-cluster-management.io/v1
kind: ManifestWork
metadata:
  name: mw-02
  namespace: ocm2
spec:
  workload:
    manifests:
      - apiVersion: v1
        kind: Pod
        metadata:
          name: hello
          namespace: default
        spec:
          containers:
            - name: hello
              image: busybox
              command: ["sh", "-c", 'echo "Hello, Kubernetes!" && sleep 3600']
          restartPolicy: OnFailure

```
- Apply it and check that it's applied successfully
```
ubuntu@ocmhub:~$ kubectl apply -f manifest-work.yaml 
manifestwork.work.open-cluster-management.io/mw-02 created
ubuntu@ocmhub:~$ kubectel -n ocm2 get manifestwork/mw-02
kubectel: command not found
ubuntu@ocmhub:~$ kubectl -n ocm2 get manifestwork/mw-02
NAME    AGE
mw-02   57s
ubuntu@ocmhub:~$ 
```
- On the managed cluster, check that the pod has been created
```
ubuntu@ocm2:~$ kubectl get po
NAME    READY   STATUS    RESTARTS   AGE
hello   1/1     Running   0          97s
ubuntu@ocm2:~$ 
```