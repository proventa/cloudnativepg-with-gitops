# CloudNativePG with GitOps

This repository provides a guide on how to experiment with [CloudNativePG](https://cloudnative-pg.io) on a local machine and manage PostgreSQL clusters the GitOps way using [ArgoCD](https://argo-cd.readthedocs.io/).  
It is based on the official documentation of CloudNativePG and ArgoCD, as well as on [Gabriele Bartolini’s blog](https://www.gabrielebartolini.it/articles/).  

> ⚠️ Please note that this is intended for experimental evaluation and should not be used in a production setting.

## Install Docker 
### Docker on Linux
https://docs.docker.com/engine/install
### Docker Desktop on Windows with WSL 2
https://docs.docker.com/desktop/setup/install/windows-install
### Docker Desktop on Mac
https://docs.docker.com/desktop/setup/install/mac-install


## Install kubectl
Included with Docker Desktop, otherwise  
https://kubernetes.io/docs/tasks/tools/install-kubectl-linux  

## Install the cnpg plugin for kubectl
https://cloudnative-pg.io/documentation/current/kubectl-plugin

```bash
curl -sSfL \
  https://github.com/cloudnative-pg/cloudnative-pg/raw/main/hack/install-cnpg-plugin.sh | \
  sudo sh -s -- -b /usr/local/bin
```
## Install kind
https://kind.sigs.k8s.io/docs/user/quick-start  
```bash
[ $(uname -m) = x86_64 ] && curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.27.0/kind-linux-amd64  
sudo mv ./kind /usr/local/bin/kind  
chmod 744 /usr/local/bin/kind  
```

## Create the Kubernetes CLuster
```bash
kind create cluster --config ./kind/five-nodes.yaml
```
Output:

```bash
Creating cluster "cnpg" ...
Creating cluster "kind" ...
 ✓ Ensuring node image (kindest/node:v1.32.2) 🖼 
 ✓ Preparing nodes 📦 📦 📦 📦 📦  
 ✓ Writing configuration 📜 
 ✓ Starting control-plane 🕹️ 
 ✓ Installing CNI 🔌 
 ✓ Installing StorageClass 💾 
 ✓ Joining worker nodes 🚜 
Set kubectl context to "kind-kind"
You can now use your cluster with:

kubectl cluster-info --context kind-kind

Thanks for using kind! 😊
```

## Install Helm
https://helm.sh/docs/intro/install/#from-script
```console
$ curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
$ chmod 700 get_helm.sh
$ ./get_helm.sh
```
## Install the Kubernetes Dashboard
https://kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/#deploying-the-dashboard-ui
```bash
# Add kubernetes-dashboard repository
helm repo add kubernetes-dashboard https://kubernetes.github.io/dashboard/
# Deploy a Helm Release named "kubernetes-dashboard" using the kubernetes-dashboard chart
helm upgrade --install kubernetes-dashboard kubernetes-dashboard/kubernetes-dashboard --create-namespace --namespace kubernetes-dashboard
# Create a sample user with long-lived Bearer Token for ServiceAccount
kubectl apply -f ./kube-dashboard/admin-user.yaml
# Get the token from the Secret
kubectl get secret admin-user -n kubernetes-dashboard -o jsonpath="{.data.token}" | base64 -d
# Start the command line proxy
kubectl -n kubernetes-dashboard port-forward svc/kubernetes-dashboard-kong-proxy 8443:443
```
Kubectl will make the Dashboard available at https://localhost:8443.

## Install CloudNativePG
Check for the latest version:
https://cloudnative-pg.io/documentation/current/installation_upgrade/#directly-using-the-operator-manifest

```bash
kubectl apply --server-side -f \
  https://raw.githubusercontent.com/cloudnative-pg/cloudnative-pg/release-1.25/releases/cnpg-1.25.1.yaml
```

Check the Deployment status:
```bash
kubectl get deployment -n cnpg-system cnpg-controller-manager
```

## Create the PostgreSQL Cluster
```bash
kubectl apply -f ./cnpg/cluster-example.yaml
```

Check the status of the pods:
```bash
kubectl get pods -w
```

Check the cluster:
```bash
kubectl get cluster cluster-example
```

Check the status of the cluster with the cnpg plugin
```bash
kubectl cnpg status cluster-example
```

Check the components:
```bash
# Pods
kubectl get pods
# Nodes
kubectl get nodes
# Custom Resource Definitions
kubectl get crds 
# Namespaces
kubectl get namespaces
# Services
kubectl get svc
# Persistent Volume Claims
kubectl get pvc
# Secrets
kubectl get secret
```

## Install ArgoCD and Argo CD CLI
https://argo-cd.readthedocs.io/en/latest/getting_started/

```bash
# ArgoCD
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

```bash
# Argo CD CLI
curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd
rm argocd-linux-amd64
```
## Login to ArgoCD UI and CLI
Port Forwarding to connect to the API and UI:
```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Get the admin password:
```bash
argocd admin initial-password -n argocd
# or
kubectl get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 --decode && echo
```

The API server and UI can then be accessed using https://localhost:8080 with username admin and the password from above.

Optional: Login Using The CLI¶

```bash
argocd login https://localhost:8080 
```

## Create and Sync an ArgoCD App
Create an simple app: 
```bash
kubectl apply -f ./argocd-apps/cnpg.yaml
```
The ArgoCD application should be now visible in the UI, but without the CNPG cluster. The cluster will be deployed with the first sync.
By default the automated sync is turned off. Use the "automated" attribute under syncPolicy to turn it on.

Sync the manifest of the cluster manually:
  - Push "Sync" in the UI.
  - Sync from local manifests directly, only for development purposes:
```bash
argocd app sync cnpg --local ./argocd-apps/cnpg.yml
```
## Test serviceTemplate with LoadBalancer 
To test cnpg serviceTemplate with LoadBalancer and expose Postgres outside the kubernetes cluster like in the manifest cnpg/cluster-example-lb.yaml

### Install the cloude-provider-kind
To test cnpg serviceTemplate with LoadBalancer and expose Postgres outside the kubernetes cluster like in the manifest cnpg/cluster-example-lb.yaml

https://github.com/kubernetes-sigs/cloud-provider-kind?tab=readme-ov-file#install

```bash
go install sigs.k8s.io/cloud-provider-kind@latest
```
### Start the cloude-provider-kind
On macOS and WSL2 you must run cloud-provider-kind using sudo
```bash
sudo ~/go/bin/cloud-provider-kind
```

### Connect to PostgreSQL from host
Get the app user password:
```bash
kubectl get secret cluster-example-app -o jsonpath='{.data.password}' | base64 --decode && echo
```
Use the IP provided by the cloud-provider-kind to connect with psql:
```bash
psql -h 172.19.0.7 -p 5432 -U app -d app
```