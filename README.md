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

# Install ArgoCD in k8s
$ kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

#Access ArgoCD UI through port-forwarding
$ kubectl get svc -n argocd
...
argocd-server           ClusterIP   10.96.227.84     <none>        80/TCP,443/TCP               35h
...
$ kubectl port-forward svc/argocd-server -n argocd 8080:443

# login with admin user and below token (as in documentation):
$ kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 --decode && echo
# you can change and delete init password

# Example Set default namespace 
#$ kubectl config set-context $(kubectl config current-context) --namespace=argocd
```
> Install ArgoCD https://argo-cd.readthedocs.io/en/stable/getting_started/
# Step 3 - Create Kubernetes deployment,service and Apply ArgoCD configulation and Access ArgoCD UI
> view ArgoCD application : myapp-argo-application
#### 3.1 Create Kubernetes Declarative
```bash
$ cd gitops-argocd
$ mkdir app-config
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
        image: kittisuw/argocd-app:1.1
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
#### 3.2 add ArgoCD configuration
```bash
$ cd gitops-argocd
$ mkdir argo-cd
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
    path: app-config/base
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
```bash
# 1.Login ArgoCD user: admin pwd : as you get from secrete
# 2.Apply ArgoCD configulation file
$ cd gitops-argocd
$ kubectl apply -f argo-cd/application.yaml
# 3.Check Argo application : myapp-argo-application
```
# Step 4 - Testing and view behavior at ArgoCD
```bash
# 1.Test edit version
$ vi deployments/deployment.yaml 
...
image: kittisuw/argocd-app:1.0 #Edit to 1.1 or 1.2
...

# 2.Test rename deployment name
...
$ vi delployments/deployment.yaml
...
metadata:
  name: myapp #Change to myapp-deployment
...

# 3.Edit deployment replicas from 2 to 4 @cluster
$ kubectl edit deploy myapp -n myapp
...
replicas: 2 #Change to 4
...

# 4. Edit selfHeal: false and try to edit replicas
$ vi argo-cd/application.yaml
...
selfHeal: true #Change to false
...
$ kubectl apply -f argocd/application.yaml
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