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
- To delete the work, simply running kubectl delete -f filename does not work
- Instead, use clusteradm delete work command
```
clusteradm delete work hello --cluster ocm2
clusteradm get works --cluster ocm2
```

### Using Placement API 
- We're going to use 'satcu' namespace
```
ubuntu@ocmhub:~$ kubectl get ns
NAME                          STATUS   AGE
default                       Active   2d3h
kube-node-lease               Active   2d3h
kube-public                   Active   2d3h
kube-system                   Active   2d3h
ocm1                          Active   2d2h
ocm2                          Active   2d1h
open-cluster-management       Active   2d3h
open-cluster-management-hub   Active   2d3h
satcu                         Active   61m
ubuntu@ocmhub:~$ 
```
- Using clusteradm, move ocm1 and ocm2 k3s clusters to satcu clusterset
```
ubuntu@ocmhub:~$ clusteradm clusterset set satcu --clusters ocm1,ocm2
Cluster ocm1 is set, from ClusterSet default to Clusterset satcu
Cluster ocm2 is set, from ClusterSet default to Clusterset satcu

ubuntu@ocmhub:~$ clusteradm get clustersets
<ManagedClusterSet> 
└── <default> 
│   ├── <BoundNamespace> 
│   ├── <Status> No ManagedCluster selected
│   ├── <Clusters> []
└── <global> 
│   ├── <BoundNamespace> 
│   ├── <Status> 2 ManagedClusters selected
│   ├── <Clusters> [ocm1 ocm2]
└── <satcu> 
    └── <BoundNamespace> satcu
    └── <Status> 2 ManagedClusters selected
    └── <Clusters> [ocm1 ocm2]
ubuntu@ocmhub:~$ 
```
- Next, we're going to create managedclustersetbinding
```
ubuntu@ocmhub:~$ cat binding.yaml 
apiVersion: cluster.open-cluster-management.io/v1beta2
kind: ManagedClusterSetBinding
metadata:
  namespace: satcu
  name: satcu
spec:
  clusterSet: satcu
ubuntu@ocmhub:~$ 

ubuntu@ocmhub:~$ kubectl apply -f binding.yaml 
managedclustersetbinding.cluster.open-cluster-management.io/satcu created
ubuntu@ocmhub:~$ 
ubuntu@ocmhub:~$ kubectl get managedclustersetbinding -n satcu
NAME    AGE
satcu   11s
ubuntu@ocmhub:~$ 
```
- Then, create a placement CRD
```
ubuntu@ocmhub:~$ cat placement.yaml 
apiVersion: cluster.open-cluster-management.io/v1beta1
kind: Placement
metadata:
  name: placement1
  namespace: satcu
spec:
  numberOfClusters: 2
  clusterSets:
    - satcu

ubuntu@ocmhub:~$ kubectl apply -f placement.yaml 
placement.cluster.open-cluster-management.io/placement1 created

ubuntu@ocmhub:~$ kubectl get placement -A
NAMESPACE   NAME         SUCCEEDED   REASON                  SELECTEDCLUSTERS
satcu       placement1   True        AllDecisionsScheduled   2
ubuntu@ocmhub:~$ 
```
- Next, we're going to submit a workload using that placement CRD
```
ubuntu@ocmhub:~$ cat work.yaml 
apiVersion: v1
kind: ServiceAccount
metadata:
  namespace: deposit
  name: my-sa
---
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: deposit
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      serviceAccountName: my-sa
      containers:
        - name: nginx
          image: nginx:1.14.2
          ports:
            - containerPort: 80

ubuntu@ocmhub:~$ clusteradm create work my-work -f work.yaml --placement satcu/placement1
create work my-work in cluster ocm2
create work my-work in cluster ocm1
ubuntu@ocmhub:~$ 
```
- Check that the workload started on ocm1 and ocm2 clusters
```
ubuntu@ocm1:~$ kubectl get po -n deposit
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-86bc46bf8f-cq67f   1/1     Running   0          113s
nginx-deployment-86bc46bf8f-j62s8   1/1     Running   0          113s
nginx-deployment-86bc46bf8f-vzqq7   1/1     Running   0          113s
ubuntu@ocm1:~$ 

ubuntu@ocm2:~$ kubectl get po -n deposit
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-86bc46bf8f-6cn9h   1/1     Running   0          102s
nginx-deployment-86bc46bf8f-kk8sb   1/1     Running   0          102s
nginx-deployment-86bc46bf8f-z9tmg   1/1     Running   0          102s
ubuntu@ocm2:~$ 
```
- Modify the placement CRD to reduce the number of cluster to 1
```
ubuntu@ocmhub:~$ kubectl -n satcu patch placement placement1 --patch '{"spec": {"clusterSets": ["satcu"],"numberOfClusters": 1}}' --type=merge
placement.cluster.open-cluster-management.io/placement1 patched
```
- That still does not actually modify the workload, since only the placement CRD was modified, and a decision was already made during work submission
```
ubuntu@ocmhub:~$ kubectl get manifestwork -A
NAMESPACE   NAME      AGE
ocm1        my-work   15m
ocm2        my-work   15m
```
- To apply the changes, use clusteradm to overwrite the same work
```
ubuntu@ocmhub:~$ clusteradm create work my-work -f work.yaml --placement satcu/placement1 --overwrite
delete work my-work in cluster ocm2
ubuntu@ocmhub:~$ 
```
- Now there's only one manifestwork, running on ocm1
```
ubuntu@ocmhub:~$ kubectl get manifestwork -A
NAMESPACE   NAME      AGE
ocm1        my-work   16m
ubuntu@ocmhub:~$ 
ubuntu@ocmhub:~$ 

ubuntu@ocm1:~$ kubectl get po -n deposit
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-86bc46bf8f-cq67f   1/1     Running   0          16m
nginx-deployment-86bc46bf8f-j62s8   1/1     Running   0          16m
nginx-deployment-86bc46bf8f-vzqq7   1/1     Running   0          16m
ubuntu@ocm1:~$ 

ubuntu@ocm2:~$ kubectl get po -n deposit
No resources found in deposit namespace.
ubuntu@ocm2:~$ 

ubuntu@ocmhub:~$ clusteradm get works --cluster ocm1
<ManifestWork> 
└── <ocm1> 
    └── <my-work> 
        └── <Number of Manifests> 2
        └── <Applied> True
        └── <Available> True
        └── <Resources> 
            └── <deployments> 
            │   ├── <deposit/nginx-deployment> applied
            └── <serviceaccounts> 
                └── <deposit/my-sa> applied
```
- Let's clean it up now
```
ubuntu@ocmhub:~$ clusteradm delete work my-work --cluster ocm1
work my-work in cluster ocm1 is deleted
ubuntu@ocmhub:~$ kubectl get manifestworks -A
No resources found
ubuntu@ocmhub:~$ 

```
- The namespace for placement and managedclustersetbinding can be changed to other namespace (probably for RBAC?)
- Change the binding.yaml and placement.yaml to use default namespace 
```
ubuntu@ocmhub:~$ vi binding.yaml 
ubuntu@ocmhub:~$ vi placement.yaml 

ubuntu@ocmhub:~$ grep -i namespace binding.yaml 
  namespace: default
ubuntu@ocmhub:~$ grep -i namespace placement.yaml 
  namespace: default
ubuntu@ocmhub:~$ 
```
- Recreate the managedclustersetbinding and placement
```
ubuntu@ocmhub:~$ kubectl apply -f binding.yaml 
managedclustersetbinding.cluster.open-cluster-management.io/satcu created
ubuntu@ocmhub:~$ kubectl get managedclustersetbinding -A
NAMESPACE   NAME    AGE
default     satcu   7s
ubuntu@ocmhub:~$ kubectl apply -f placement.yaml 
placement.cluster.open-cluster-management.io/placement1 created
ubuntu@ocmhub:~$ 
ubuntu@ocmhub:~$ 
ubuntu@ocmhub:~$ kubectl get placement
NAME         SUCCEEDED   REASON                  SELECTEDCLUSTERS
placement1   True        AllDecisionsScheduled   2
ubuntu@ocmhub:~$ 
```
- Resubmit work using clusteradm
```
ubuntu@ocmhub:~$ clusteradm create work work-1 -f work.yaml --placement default/placement1
create work work-1 in cluster ocm1
create work work-1 in cluster ocm2
ubuntu@ocmhub:~$ 
ubuntu@ocmhub:~$ kubectl get manifestwork -A
NAMESPACE   NAME     AGE
ocm1        work-1   17s
ocm2        work-1   17s
ubuntu@ocmhub:~$ kubectl get placementdecision
NAME                    AGE
placement1-decision-1   2m35s
ubuntu@ocmhub:~$ kubectl get placementdecision placement1-decision-1 -o yaml
apiVersion: cluster.open-cluster-management.io/v1beta1
kind: PlacementDecision
metadata:
  creationTimestamp: "2024-11-30T18:14:20Z"
  generation: 1
  labels:
    cluster.open-cluster-management.io/decision-group-index: "0"
    cluster.open-cluster-management.io/decision-group-name: ""
    cluster.open-cluster-management.io/placement: placement1
  name: placement1-decision-1
  namespace: default
  ownerReferences:
  - apiVersion: cluster.open-cluster-management.io/v1beta1
    blockOwnerDeletion: true
    controller: true
    kind: Placement
    name: placement1
    uid: a8f8b87c-14d6-4e8a-9e79-9c682bf94afa
  resourceVersion: "31162"
  uid: 4cca8131-f74c-4cde-8767-e503df3cfea7
status:
  decisions:
  - clusterName: ocm1
    reason: ""
  - clusterName: ocm2
    reason: ""
ubuntu@ocmhub:~$
```
- Check on the managed clusters that the workload started
```
ubuntu@ocm1:~$ kubectl get po -n deposit
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-86bc46bf8f-94whj   1/1     Running   0          8m14s
nginx-deployment-86bc46bf8f-bvlgb   1/1     Running   0          8m14s
nginx-deployment-86bc46bf8f-g986z   1/1     Running   0          8m14s
ubuntu@ocm1:~$ 

ubuntu@ocm2:~$ kubectl get po -n deposit
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-86bc46bf8f-5hvpm   1/1     Running   0          8m18s
nginx-deployment-86bc46bf8f-drzqb   1/1     Running   0          8m18s
nginx-deployment-86bc46bf8f-fsr8g   1/1     Running   0          8m18s
ubuntu@ocm2:~$ 
```
- Let's clean it up
```
ubuntu@ocmhub:~$ clusteradm get works --clusters ocm1,ocm2
<ManifestWork> 
└── <ocm1> 
│   ├── <work-1> 
│       └── <Number of Manifests> 2
│       └── <Applied> True
│       └── <Available> True
│       └── <Resources> 
│           └── <serviceaccounts> 
│           │   ├── <deposit/my-sa> applied
│           └── <deployments> 
│               └── <deposit/nginx-deployment> applied
└── <ocm2> 
    └── <work-1> 
        └── <Number of Manifests> 2
        └── <Applied> True
        └── <Resources> 
        │   ├── <serviceaccounts> 
        │   │   ├── <deposit/my-sa> applied
        │   ├── <deployments> 
        │       └── <deposit/nginx-deployment> applied
        └── <Available> True

ubuntu@ocmhub:~$ clusteradm delete work work-1 --clusters ocm1,ocm2
work work-1 in cluster ocm1 is deleted
work work-1 in cluster ocm2 is deleted
ubuntu@ocmhub:~$ 

```
