# Step 0 - Install minikube for local kubernetes cluster
```bash
$ curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-darwin-amd64
$ sudo install minikube-darwin-amd64 /usr/local/bin/minikube
$ minikube start
#Check context
$ kubectl config get-contexts
#Set context to minikube
$ kubectl config use-context minikube
# clone project
$ git clone git@github.com:kittisuw/gitops-argocd.git
```
# Step 1 — Install ArgoCD
```bash
# Create namespace for install Argo
$ kubectl create namespace argocd

# Install ArgoCD in k8s
$ kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Access ArgoCD UI
$ kubectl get svc -n argocd
...
argocd-server           ClusterIP   10.96.227.84     <none>        80/TCP,443/TCP               35h
...
$ kubectl port-forward svc/argocd-server -n argocd 8080:443

# login with admin user and below token (as in documentation):
$ kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 --decode && echo
# you can change and delete init password
# Example Set default namespace $ kubectl config set-context $(kubectl config current-context) --namespace=argocd
```
# Step 2 - Apply ArgoCR configulation file
```bash
# 1.Login ArgoCD user: admin pwd : as you get from secrete
# 2.Apply ArgoCD configulation file
$ cd gitops-argocd
$ kubectl apply -f argo-cd/application.yaml
# 3.Check Argo application : myapp-argo-application
```
# Step 3 - Testing and view behavior at ArgoCD
```bash
# 1.Test edit version to 1.0 or 1.1 or 1.2 and Check Argo application : myapp-argo-application
vi deployments/deployment.yaml 
...
image: kittisuw/argocd-app:1.1
...

# 2.Test rename deployment name and Check Argo application : myapp-argo-application
...
vi delployments/deployment.yaml
...
metadata:
  name: myapp #Change to myapp-deployment
...

# 3.Edit deployment replicas from 2 to 4 @cluster and Check Argo application : myapp-argo-application
kubectl edit deploy myapp -n myapp
...
replicas: 2 #Change to 4
...

# 4. Edit selfHeal: false and try to edit replicas
$ vi argo-cd/application.yaml
...
selfHeal: false
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
> Install ArgoCD https://argo-cd.readthedocs.io/en/stable/getting_started/