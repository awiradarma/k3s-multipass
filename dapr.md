# Install dapr using helm 
From https://docs.dapr.io/operations/hosting/kubernetes/kubernetes-deploy/

```
ubuntu@master:~$ helm repo add dapr https://dapr.github.io/helm-charts/
"dapr" has been added to your repositories
ubuntu@master:~$ helm repo update
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "dapr" chart repository
...Successfully got an update from the "bitnami" chart repository
Update Complete. ⎈Happy Helming!⎈
ubuntu@master:~$ helm upgrade --install dapr dapr/dapr \
--version=1.15 \
--namespace dapr-system \
--create-namespace \
--wait
Release "dapr" does not exist. Installing it now.
NAME: dapr
LAST DEPLOYED: Sat Apr 12 20:58:18 2025
NAMESPACE: dapr-system
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
Thank you for installing Dapr: High-performance, lightweight serverless runtime for cloud and edge

Your release is named dapr.

To get started with Dapr, we recommend using our quickstarts:
https://github.com/dapr/quickstarts

For more information on running Dapr, visit:
https://dapr.io
> Warning: The default storage size for the Scheduler is 1Gi, which may not be sufficient for production deployments.
> You may want to consider reinstalling Dapr with a larger Scheduler storage of at least 16Gi.
>
> --set dapr_scheduler.cluster.storageSize=16Gi
>
> For more information, see https://docs.dapr.io/operations/hosting/kubernetes/kubernetes-persisting-scheduler

```

## Check installation

```
ubuntu@master:~$ kubectl get po -n dapr-system
NAME                                     READY   STATUS    RESTARTS   AGE
dapr-operator-69846b7b96-bltls           1/1     Running   0          38s
dapr-placement-server-0                  1/1     Running   0          38s
dapr-scheduler-server-0                  1/1     Running   0          38s
dapr-scheduler-server-1                  1/1     Running   0          37s
dapr-scheduler-server-2                  1/1     Running   0          37s
dapr-sentry-6fb4c68b8-ftcj8              1/1     Running   0          38s
dapr-sidecar-injector-68d54fdf54-664fh   1/1     Running   0          38s
ubuntu@master:~$ 
```

## Install dapr dashboard
```
ubuntu@master:~$ helm install dapr-dashboard dapr/dapr-dashboard --namespace dapr-system
NAME: dapr-dashboard
LAST DEPLOYED: Sat Apr 12 21:06:09 2025
NAMESPACE: dapr-system
STATUS: deployed
REVISION: 1
TEST SUITE: None
ubuntu@master:~$ 

ubuntu@master:~$ kubectl get po -n dapr-system
NAME                                     READY   STATUS    RESTARTS   AGE
dapr-dashboard-789795555d-vsxdt          1/1     Running   0          85s
dapr-operator-69846b7b96-bltls           1/1     Running   0          9m15s
dapr-placement-server-0                  1/1     Running   0          9m15s
dapr-scheduler-server-0                  1/1     Running   0          9m15s
dapr-scheduler-server-1                  1/1     Running   0          9m14s
dapr-scheduler-server-2                  1/1     Running   0          9m14s
dapr-sentry-6fb4c68b8-ftcj8              1/1     Running   0          9m15s
dapr-sidecar-injector-68d54fdf54-664fh   1/1     Running   0          9m15s
ubuntu@master:~$ 

ubuntu@master:~$ helm list -n dapr-system
NAME          	NAMESPACE  	REVISION	UPDATED                                	STATUS  	CHART                	APP VERSION
dapr          	dapr-system	1       	2025-04-12 20:58:18.258759765 -0500 CDT	deployed	dapr-1.15.4          	1.15.4     
dapr-dashboard	dapr-system	1       	2025-04-12 21:06:09.704818243 -0500 CDT	deployed	dapr-dashboard-0.15.0	0.15.0     
ubuntu@master:~$ 

```