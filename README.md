# GitOps w/ ArgoCD a Tutorial
# Step 0 - Prereqs
#### 0.1 - Docker Desktop
>Install Docker Desktop: https://docs.docker.com/engine/install
#### 0.2 - Minikube
```bash
#Install minikube for local kubernetes cluster
$ curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-darwin-amd64
$ sudo install minikube-darwin-amd64 /usr/local/bin/minikube
$ minikube start
##Check context
$ kubectl config get-contexts
##Set context to minikube
$ kubectl config use-context minikube
```
> Get started minikube https://minikube.sigs.k8s.io/docs/start/
#### 0.3 - Public/Private image registry
> in this demo I using dockerhub : https://hub.docker.com
#### 0.4 - Repo used for tutorial
```bash
##clone this repository for demo
$ git clone git@github.com:kittisuw/gitops-argocd.git
```
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
# 1.Test edit version
$ vi deployment.yaml 
...
image: kittisuw/argocd-app:1.0 #Edit to 1.0 or 1.1
...
```
#### 4.2 - Rename Kubernetes deployment 
```bash
...
$ vi deployment.yaml
...
metadata:
  name: myapp #Change to myapp-deployment
...
```
#### 4.3 - Edit on the fire Kubernetes ReplicaSet from 2 to 4
```bash
$ kubectl edit deploy myapp -n myapp
...
replicas: 2 #Change to 4
...
```
#### 4.4 Edit selfHeal: false and try to edit replicas
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
#Stop minikube cluster
minikube stop
```
### Learning Resorce
> https://www.youtube.com/watch?v=5gsHYdiD6v8   
> https://www.youtube.com/watch?v=MeU5_k9ssrs&t=2244s   
> https://www.youtube.com/watch?v=571cbVNahpE   
> Kustomize: https://github.com/kubernetes-sigs/kustomize   
> Add webhook: https://youtu.be/LhsnaeOGC-g