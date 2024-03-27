# ArgoCD Setup

In this lab we will get ArgoCD ready for our school app.

## Install ArgoCD

### Install the ArgoCD Server

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Wait until all the pods are running

```bash
watch kubectl get po -n argocd
```

### Install the ArgoCD Client (Optional)

For details on installing the client, [refer to this document.](https://argo-cd.readthedocs.io/en/stable/cli_installation/)

For linux can run the following:

```bash
sudo curl -sSL -o /usr/local/bin/argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
sudo chmod +x /usr/local/bin/argocd
```

## Access ArgoCD

### Expose the ArgoCD API Server

```bash
kubectl port-forward svc/argocd-server -n argocd 8002:443
```

### Get the admin password

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```

For this lab, I won't change the password, but you should go ahead and change the password and remove the K8s secret.

### Login using the CLI and/or the UI

To login using the CLI:

```bash
argocd login 127.0.0.1:8002
```

Answer `y` to the cert validation warning, then use
username: admin
password: <THE_PASSWORD_YOU_GOT_ABOVE>

You can also log in to the UI by opening a browser window and going to https://127.0.0.1:8002

> This concludes this lab

q7vFXtgEPeE2yz5n