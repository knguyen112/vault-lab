# Prepare Vault for the School App

In this lab we'll prepare Vault to serve the MongoDB credentials to the API.

## Port Forward the Vault Service

First make sure you are port forwarding the Vault service in another terminal.

```bash
kubectl port-forward service/vault 8200:8200
```

## Login to Vault

Let's now log in to Vault

```bash
# export the vault address and login
export VAULT_ADDR=http://127.0.0.1:8200
vault login <ROOT_TOKEN>
# Example:
vault login s.SBGKz0bx4USd6k4ExuhLUjYC
```

## Add mongoDB Credentials
We will add a static KV secret for the school app. This secret is the mongoDB credentials so that the API could connect to mongoDB.

```bash
vault secrets enable -version=2 -path=internal kv
vault kv put internal/schoolapp/mongodb schoolapp_DB_USERNAME=schoolapp schoolapp_DB_PASSWORD=mongoRootPass
```

You can check the credentials in the UI or the CLI by running:

```bash
vault kv get internal/schoolapp/mongodb
```

## Create a School App Policy

```bash
vault policy write schoolapp ./schoolapp.hcl
```

## Setup K8s Auth Method in Vault

```bash
# Enable K8s auth method
vault auth enable kubernetes
# Configure the K8s auth method (run this command from within the pod)
kubectl exec -it pod/vault-0 -- sh
# export the vault address and login
export VAULT_ADDR=http://127.0.0.1:8200
vault login <ROOT_TOKEN>
vault write auth/kubernetes/config \
    issuer="https://kubernetes.default.svc.cluster.local" \
    token_reviewer_jwt="$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
    kubernetes_host="https://$KUBERNETES_PORT_443_TCP_ADDR:443" \
    kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt
vault write auth/kubernetes/role/schoolapp \
    bound_service_account_names=schoolapp \
    bound_service_account_namespaces=schoolapp \
    policies=schoolapp \
    ttl=5s \
    max_ttl=20s
```

> This concludes this lab