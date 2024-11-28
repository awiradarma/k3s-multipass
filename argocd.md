## Exposing ArgoCD using traefik
- Disable TLS by editing the configmap and add server.insecure: "true"
- Restart argocd-server
- Create a Traefik ingressroute
```
$ kubectl edit cm argocd-cmd-params-cm -n argocd
$ kubectl get cm argocd-cmd-params-cm -n argocd -o yaml
apiVersion: v1
data:
  server.insecure: "true"
kind: ConfigMap
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","kind":"ConfigMap","metadata":{"annotations":{},"labels":{"app.kubernetes.io/name":"argocd-cmd-params-cm","app.kubernetes.io/part-of":"argocd"},"name":"argocd-cmd-params-cm","namespace":"argocd"}}
  creationTimestamp: "2024-11-23T22:50:26Z"
  labels:
    app.kubernetes.io/name: argocd-cmd-params-cm
    app.kubernetes.io/part-of: argocd
  name: argocd-cmd-params-cm
  namespace: argocd
  resourceVersion: "34937"
  uid: b55886b6-8b65-4326-bfd8-c73c63213dc8

$ kubectl rollout restart deployment argocd-server -n argocd
deployment.apps/argocd-server restarted

$ cat argocd.yaml
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: argocd-server
  namespace: argocd
spec:
  entryPoints:
    - websecure
  routes:
    - kind: Rule
      match: Host(`argocd.acme.com`)
      priority: 10
      services:
        - name: argocd-server
          port: 80
    - kind: Rule
      match: Host(`argocd.acme.com`) && Header(`Content-Type`, `application/grpc`)
      priority: 11
      services:
        - name: argocd-server
          port: 80
          scheme: h2c
  tls:
    certResolver: default


```
## Using ApplicationSet to deploy to multiple clusters

- Set up an argocd project with the right set of destination clusters, namespace and source repo

![List of clusters](clusters.png)

![ArgoCD Project](project.png)

- Sample ApplicationSet definition
```
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: guestbook
spec:
  goTemplate: true
  goTemplateOptions: ["missingkey=error"]
  generators:
  - list:
      elements:
      - cluster: in-cluster
        url: https://kubernetes.default.svc
      - cluster: cluster2
        url: https://10.165.182.183:6443
  template:
    metadata:
      name: '{{.cluster}}-guestbook'
    spec:
      project: guestbook
      source:
        repoURL: https://github.com/argoproj/argocd-example-apps.git
        targetRevision: HEAD
        path: guestbook
      destination:
        server: '{{.url}}'
        namespace: default

```

- How to create an application set
```
ubuntu@master:~$ argocd appset create applicationset.yaml 
ApplicationSet 'guestbook' created
Name:               argocd/guestbook
Project:            guestbook
Server:             {{.url}}
Namespace:          default
Source:
- Repo:             https://github.com/argoproj/argocd-example-apps.git
  Target:           HEAD
  Path:             guestbook
SyncPolicy:         <none>

```

![Applications](applications.png)
