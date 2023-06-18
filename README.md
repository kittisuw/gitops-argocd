# GitOps w/ ArgoCD a Tutorial
# Step 0 - Prereqs
#### 0.1 - [Kind](https://github.com/kittisuw/Kubernetes/tree/main/local-development/kind) for running a Kubernetes cluster locally, using Docker containers as cluster nodes
#### 0.2 - Public/Private image registry
> in this demo I using dockerhub : https://hub.docker.com
#### 0.3 - Repo used for tutorial
```bash
├── app #Replace with app repository Dev is owner of this repo
│   ├── Dockerfile
│   ├── index.js
│   └── package.json
├── app-config #Replace with app-config repository DevOps is owner of this repo
    ├── application.yaml #ArgoCD application config
    ├── deployment.yaml #Kubernetes deployment config
    └── service.yaml #Kubernetes service config
```
> ArgoCD best practice: https://argo-cd.readthedocs.io/en/stable/user-guide/best_practices/
# Step 1 - Create Simple Node.js application for demo
```bash
$ cd app
#Buid an tag application
$ docker build . -t kittisuw/argocd-app:1.0
$ docker build . -t kittisuw/argocd-app:1.1
$ docker build . -t kittisuw/argocd-app:1.2
#Push to image registry In this demo I using dockerhub
$ docker push kittisuw/argocd-app:1.0
$ docker push kittisuw/argocd-app:1.1
$ docker push kittisuw/argocd-app:1.2
```
# Step 2 — Install ArgoCD on Kubernetes cluster
```bash
# Create namespace for install Argo
$ kubectl create namespace argocd
$ kubectl get ns
...
NAME              STATUS   AGE
argocd            Active   8s
...

# Install ArgoCD in k8s
$ kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Waiting all pod status is Running
$ kubectl get po -n argocd                                                               ok  minikube/myapp kube  15:48:53 
NAME                                      READY   STATUS    RESTARTS   AGE
pod/argocd-application-controller-0       1/1     Running   0          49s
pod/argocd-dex-server-6c55787bc6-5mzv2    1/1     Running   0          50s
pod/argocd-redis-74d8c6db65-b25sr         1/1     Running   0          50s
pod/argocd-repo-server-6c44847cf9-tzclb   1/1     Running   0          50s
pod/argocd-server-67b65559fb-ffqkb        0/1     Running   0          49s
... 

#Get Argocd admin init password from secret
$ kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 --decode && echo
# you can change and delete init password

#Access ArgoCD UI through port-forwarding
$ kubectl get svc -n argocd
...
argocd-server           ClusterIP   10.96.227.84     <none>        80/TCP,443/TCP               35h
...
$ kubectl port-forward svc/argocd-server -n argocd 8080:443
```
>Example Set default namespace    
> $ kubectl config set-context $(kubectl config current-context) --namespace=argocd  
 
> Install ArgoCD https://argo-cd.readthedocs.io/en/stable/getting_started/
# Step 3 - Create Kubernetes deployment,service and ArgoCD application config and Access ArgoCD UI
#### 3.1 - Create Kubernetes Declarative
```bash
$ cd gitops-argocd/app-config
$ vi deployment.yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  selector:
    matchLabels:
      app: myapp
  replicas: 2
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: kittisuw/argocd-app:1.0
        ports:
        - containerPort: 8080
---
$ vi service.yaml
---
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
spec:
  selector:
    app: myapp
  ports:
  - port: 8080
    protocol: TCP
    targetPort: 8080
---
```
#### 3.2 - Add ArgoCD configuration
```bash
$ cd gitops-argocd/app-config
$ vi application.yaml
---
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-argo-application
  namespace: argocd
spec:
  project: default

  source:
    repoURL: https://github.com/kittisuw/gitops-argocd.git
    targetRevision: HEAD
    path: app-config
  destination: 
    server: https://kubernetes.default.svc
    namespace: myapp

  syncPolicy:
    syncOptions:
    - CreateNamespace=true

    automated:
      selfHeal: true
      prune: true
---
```
#### 3.3 - Apply ArgoCD configuration
```bash
#Apply ArgoCD configulation file
$ kubectl apply -f application.yaml
```
#### 3.4 - Login ArgoCD https://localhost:8080 user: admin pwd : as you get from secrete and check Argo application : myapp-argo-application
# Step 4 - Testing scenario and view behavior at ArgoCD
#### 4.1 - Update image version
```bash
$ vi deployment.yaml 
...
image: kittisuw/argocd-app:1.0 #Edit to 1.0 or 1.1
...
$ {git add .;git commit -m 'Update deployment';git push origin master} 
```
#### 4.2 - Rename Kubernetes deployment 
```bash
...
$ vi deployment.yaml
...
metadata:
  name: myapp #Change to myapp-deployment
...
$ {git add .;git commit -m 'Update deployment';git push origin master} 
```
#### 4.3 - Edit on the fire Kubernetes ReplicaSet from 2 to 4
```bash
$ kubectl edit deploy myapp -n myapp
...
replicas: 2 #Change to 4
...
$ {git add .;git commit -m 'Update deployment';git push origin master}
```
#### 4.4 Edit ArgoCD application config selfHeal: false and try to edit replicas
```bash
$ vi argo-cd/application.yaml
...
selfHeal: true #Change to false
...
$ kubectl apply -f application.yaml
$ k edit deploy myapp -n myapp
...
replicas: 2 #Change to 4
...
```
# Step 5 - Advance
#### 5.1 - Update realtime desire state with webhook
By default ArgoCD will pull desire state from git repository every 3 minutes if you would like to force update realtime just set webhook to https://{argoCD}/api/webhook
> Add webhook: https://youtu.be/LhsnaeOGC-g
#### 5.2 - ArgoCD intregate with kustomize overlay
Kustomize is Kubernetes native configuration management just patch some config without change base yaml and using overlay technic for patch different environment without sepratate config branch (In this demo I using only master branch) Currenly ArgoCD are fully support kustomize.
> Offial Kustomize site : https://kustomize.io 
```bash
$ cd app-config
$ mkdir argo-cd base overlays
$ mv application.yaml ./argo-cd
$ mv deployment.yaml service.yaml ./base

# Add kuztomize config 
$ cd base
vi kustomization.yaml
---
resources:
  - deployment.yaml
  - service.yaml
---

# Add folder for apply different config for each environment
$ mkdir overlays
$ cd overlays
$ mkdir dev stage prod

# Add config for each environment for test pach image version
# ArgoCD will watching kustomiation.yaml
# repeat this step to prod, stage folder
$ cd dev
$ vi kustomization.yaml
---
bases:
  - ../../base
patches:
  - image.yaml
---
# patch image version dev -> latest , stage -> 1.1 , prod 1.0
$ vi image.yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      containers:
      - name: myapp
        image: kittisuw/argocd-app:latest  #dev -> latest , stage -> 1.1 , prod 1.0
---
# Deelete ArgoCD application config
$ kubectl delete -f app-config/argo-cd/application.yaml

# Create ArgoCD application for each environment
# For this demo I using kubernetes 1 cluster 3 environment by separate namespace
# If You are using separate cluster read ArgoCD application set
$ cd app-config/argo-cd
$ vi application-dev.yaml
---
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-argo-application-dev
  namespace: argocd
spec:
  project: default

  source:
    repoURL: https://github.com/kittisuw/gitops-argocd.git
    targetRevision: HEAD
    path: app-config/overlays/dev
  destination: 
    server: https://kubernetes.default.svc
    namespace: dev

  syncPolicy:
    syncOptions:
    - CreateNamespace=true

    automated:
      selfHeal: true
      prune: true
---
$ vi application-stage.yaml
---
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-argo-application-stage
  namespace: argocd
spec:
  project: default

  source:
    repoURL: https://github.com/kittisuw/gitops-argocd.git
    targetRevision: HEAD
    path: app-config/overlays/stage
  destination: 
    server: https://kubernetes.default.svc
    namespace: stage

  syncPolicy:
    syncOptions:
    - CreateNamespace=true

    automated:
      selfHeal: true
      prune: true
---
vi application-prod.yaml
---
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-argo-application-prod
  namespace: argocd
spec:
  project: default

  source:
    repoURL: https://github.com/kittisuw/gitops-argocd.git
    targetRevision: HEAD
    path: app-config/overlays/prod
  destination: 
    server: https://kubernetes.default.svc
    namespace: prod

  syncPolicy:
    syncOptions:
    - CreateNamespace=true

    automated:
      selfHeal: true
      prune: true
---
# Apply ArgoCD application config for each environment
$ kubectl apply -f application-dev.yaml -f application-stage.yaml -f application-prod.yaml
```


# Cleanup
```bash
#Delete Argocd config
kubctl delete -f argo-cd/application.yaml
#Delete all object in namesapce myapp
kubectl delete all --all -n myapp
#Delete namespace not wait for confirmation
kubectl delete ns myapp --grace-period=0 --force
#Uninstall argocd
kubectl delete -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
#Delete namespace argocd not wait for confirmation
kubectl delete ns argocd --grace-period=0 --force
#Delete kind cluster
kind delete cluster --name {Cluster name}
```
### Learning Resorce
> https://www.youtube.com/watch?v=5gsHYdiD6v8   
> https://www.youtube.com/watch?v=MeU5_k9ssrs&t=2244s   
> https://www.youtube.com/watch?v=571cbVNahpE   
> Kustomize: https://github.com/kubernetes-sigs/kustomize   
