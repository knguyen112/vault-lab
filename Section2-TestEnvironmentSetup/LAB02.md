# Deploy the SchoolApp in K8s

In this lab we will build our K8s test environment and install the schoolapp into it.

## The Setup

For our setup we used the following

### Docker Desktop Version
```bash
$docker version
Server: Docker Desktop
 Engine:
  Version:          20.10.14
```

### K8s version
```bash
$kubectl version
Client Version: version.Info{Major:"1", Minor:"24", GitVersion:"v1.24.0", GitCommit:"4ce5a8954017644c5420bae81d72b09b735c21f0", GitTreeState:"clean", BuildDate:"2022-05-03T13:46:05Z", GoVersion:"go1.18.1", Compiler:"gc", Platform:"linux/amd64"}
Kustomize Version: v4.5.4
Server Version: version.Info{Major:"1", Minor:"24", GitVersion:"v1.24.0", GitCommit:"4ce5a8954017644c5420bae81d72b09b735c21f0", GitTreeState:"clean", BuildDate:"2022-05-03T13:38:19Z", GoVersion:"go1.18.1", Compiler:"gc", Platform:"linux/amd64"}
```

### Helm version
```bash
$helm version
version.BuildInfo{Version:"v3.4.0", GitCommit:"7090a89efc8a18f3d8178bf47d2462450349a004", GitTreeState:"clean", GoVersion:"go1.14.10"}
```

## Install the School App
```bash
# Setup MongoDB DB in K8s
kubectl create ns schoolapp
helm repo add bitnami https://charts.bitnami.com/bitnami
helm install schoolapp-mongodb --namespace schoolapp \
 --set auth.enabled=true \
 --set auth.rootUser=schoolapp \
 --set auth.rootPassword=mongoRootPass \
  bitnami/mongodb

# Add project repo to helm
helm repo add schoolapp https://gitlab.com/api/v4/projects/34240616/packages/helm/stable

# Install the Frontend
helm install frontend -n schoolapp schoolapp/schoolapp-frontend
# Install the API
helm install api -n schoolapp schoolapp/schoolapp-api
# Port forward the frontend
kubectl -n schoolapp port-forward service/frontend 8001:8080
# Port forward the api
kubectl -n schoolapp port-forward service/api 5000:5000
```

## Test the School App

From a browser, go to http://127.0.0.1:8001

> This concludes this lab