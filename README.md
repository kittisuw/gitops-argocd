# Step 0 - Install minikube for local kubernetes cluster
```bash
$ curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-darwin-amd64
$ sudo install minikube-darwin-amd64 /usr/local/bin/minikube
$ minikube start
#Check context
$ kubectl config get-contexts
#Set context to minikube
$ kubectl config use-context minikube
```
# Step 1 — Install ArgoCD
```bash
# Create namespace for install Argo
$ kubectl create namespace argocd

# Install ArgoCD in k8s
$ kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Get Argocd service name
$ kubectl get svc -n argocd
...
argocd-server           ClusterIP   10.96.227.84     <none>        80/TCP,443/TCP               35h
...

# Using minikube expose service port for access
$ minikube service argocd-server -n argocd

# login with admin user and below token (as in documentation):
$ kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 --decode && echo
# you can change and delete init password
# Example Set default namespace $ kubectl config set-context $(kubectl config current-context) --namespace=argocd
```

# Testing 
```bash
# 1.Login ArgoCD https://127.0.0.1:8080
# 2.Apply ArgoCR configulation file
$ git clone git@repo.blockfint.com:dev-sec-ops/argocd/argocd-app-config.git
$ kubectl apply -f application.yaml

# 3.Check Argo application : myapp-argo-application
# 4.Test edit version to 1.0 or 1.1 or 1.2 and Check Argo application : myapp-argo-application
vi dev/deployment.yaml 
...
image: nanajanashia/argocd-app:1.2
...
# 5.Test rename deployment name and Check Argo application : myapp-argo-application
...
vi dev/deployment.yaml
...
metadata:
  name: myapp #Change to myapp-deployment
...
# 6. Edit deployment replicas from 2 to 4 @cluster and Check Argo application : myapp-argo-application
kubectl edit deploy myapp -n myapp
```

# Cleanup 
```bash
#delete Argocd config
kubctl delete -f application.yaml
#delete all object in namesapce myapp
kubectl delete all --all -n myapp
#delete namespace
kubectl delete ns myapp
#Uninstall argocd
kubectl delete -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
#delete namespace argocd
kubectl delete ns argocd
#Stop minikube cluster
minikube stop
```
