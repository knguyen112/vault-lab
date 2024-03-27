# nseal Key 1: veH4BVRcRvwjg9bJ92ioSMkxlrh7ld2f425e2bx+QsQ=
# Initial Root Token: s.xlOckhCN6AdIkw3iwfh546Dd

# Deploy the Vault Server in K8s

In this lab we will build deploy the Vault Server in K8s via Helm.

## Install Vault via Helm

Let's install Vault with all the values defaults for the Helm chart.

```bash
kubectl create ns vault
helm repo add hashicorp https://helm.releases.hashicorp.com
helm install vault --namespace vault --set server.image.tag=1.9.3 hashicorp/vault
```

## Access Vault Server in K8s

You can exec into the vault pod and run vault commands, but it's preferable to use the vault binary running on your computer. Please make sure you have the binary or [downlaod it from HashiCorp's site.](https://www.vaultproject.io/downloads)

```bash
# Expose the Vault service
kubectl -n vault port-forward service/vault 8200:8200
# export the vault address
export VAULT_ADDR=http://127.0.0.1:8200
```

## Unseal Vault

Next we'll initialize and unseal Vault. Make sure to save your root token and unseal key

```bash
vault operator init -key-shares=1 -key-threshold=1
vault operator unseal <UNSEAL_KEY>
# Example:
vault operator unseal veH4BVRcRvwjg9bJ92ioSMkxlrh7ld2f425e2bx+QsQ=
```

## Login to Vault

Let's now log in to Vault

```bash
vault login <ROOT_TOKEN>
# Example:
vault login s.xlOckhCN6AdIkw3iwfh546Dd
```

The token information will be stored in the token helper. You do NOT need to run "vault login" again. Future Vault requests will automatically use this token.

## Verify Login

Now let's make sure we're using the root token. Double check that the policy used is `root`. 

```bash
vault token lookup
```

> This concludes this lab