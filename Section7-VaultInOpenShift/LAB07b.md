# Install Vault and ArgoCD into Openshift

In this lab we will install Vault and ArgoCD into our Openshift cluster. 

## Log in

We will continue our work from the previous lab.

Make sure you're ssh'ed into the Azure VM and run this command to enable oc commands:
```bash
eval $(crc oc-env)
oc login -u kubeadmin https://api.crc.testing:6443
```

You may need to bounce the CRC environment from time to time, run the following:

```bash
crc stop
crc start
```


## Install Helm

Let's install helm

```bash
curl https://baltocdn.com/helm/signing.asc | sudo apt-key add -
sudo apt-get install apt-transport-https --yes
echo "deb https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm
```

## Install Vault

Take a look at the values.yaml file for the [Vault helm chart](https://github.com/hashicorp/vault-helm/blob/main/values.yaml) to see the differences for OpenShift.

Run the following commands:

```bash
oc new-project vault
helm repo add hashicorp https://helm.releases.hashicorp.com
helm install vault hashicorp/vault \
  --set server.image.tag=1.9.3 \
  --set "global.openshift=true" \
  --set "server.dev.enabled=true"
```

### Configure Vault

Run these commands from within the pod):

```bash
oc exec -it pod/vault-0 -- sh
export VAULT_ADDR=http://127.0.0.1:8200
vault login root
vault secrets enable -version=2 -path=internal kv
vault kv put internal/schoolapp/mongodb schoolapp_DB_USERNAME=schoolapp schoolapp_DB_PASSWORD=mongoRootPass
vault policy write schoolapp - <<EOF
path "internal/data/schoolapp/mongodb" {
  capabilities = ["list","read"]
}
EOF
vault auth enable kubernetes
vault write auth/kubernetes/config \
    issuer="https://kubernetes.default.svc.cluster.local" \
    token_reviewer_jwt="$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
    kubernetes_host="https://$KUBERNETES_PORT_443_TCP_ADDR:443" \
    kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt
vault write auth/kubernetes/role/schoolapp \
    bound_service_account_names=schoolapp \
    bound_service_account_namespaces=schoolapp \
    policies=schoolapp \
    ttl=24h
```

Type exit to exit the vault container execution and back to the Azure VM prompt.

## Install ArgoCD

We will use the OpenShift Operator Hub provided GitOps operator, which includes an implementation of ArgoCD.

Open a browser window on your local computer and go to the following URL to access the UI
https://console-openshift-console.apps-crc.testing

login as the `kubeadmin` user and use the password that was shown in the output of the `crc start` command in the previous lab.

Go to Operators > OperatorHub and search for `openshift gitops`
and install this operator which is an implementation of ArgoCD for Openshift.

An instance of Argo CD gets installed in the openshift-gitops project/namespace

Run the following to confirm that the new Argo CD instance is running:

```bash
oc project openshift-gitops
oc get all
```

Notice that a route gets created. You can view the route in the routes section in the UI. If you followed the previous lab, you'll already have DNS record in your local computer's hosts file.

However, we won't be using this ArgoCD instance and will create a new one with the ArgoCD Vault Plugin in the final lab.

> This concludes this lab