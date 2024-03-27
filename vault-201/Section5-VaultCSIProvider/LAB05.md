# School App Example with the Vault CSI Provider

In this lab we will re-run the School App but instead of using the Vault Agent Injector, we'll use the [Vault CSI Provider.](https://www.vaultproject.io/docs/platform/k8s/csi)

## Redeploy the API Helm Chart for Vault

Let's re-install Vault but disabling the injector and enabling CSI

```bash
helm upgrade vault --namespace vault \
    --set server.image.tag=1.9.3 \
    --set injector.enabled=false \
    --set csi.enabled=true \
    hashicorp/vault
```

Notice the difference below, we now have the `vault-csi-provider` instead of the injector
```
NAME                           READY   STATUS    RESTARTS   AGE
pod/vault-0                    1/1     Running   0          24h
pod/vault-csi-provider-c65gm   1/1     Running   0          17s
```

We've already configured the schoolapp policy and K8s auth method in Lab03, but adding it here for completeness.

## Create a School App Policy

```bash
vault policy write schoolapp ./schoolapp.hcl
```

## Setup K8s Auth Method in Vault

```bash
# Enable K8s auth method
vault auth enable kubernetes
# Configure the K8s auth method (run this command from within the pod)
kubectl exec -it pod/vault-0
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


## Install the secrets store CSI driver

```bash
helm repo add secrets-store-csi-driver https://kubernetes-sigs.github.io/secrets-store-csi-driver/charts
helm install csi --namespace vault secrets-store-csi-driver/secrets-store-csi-driver
```

Check the pods are ready by running the following command:
```bash
kubectl get pods
```
**Expected Output:**
```
NAME                                 READY   STATUS    RESTARTS   AGE
csi-secrets-store-csi-driver-tkt6s   3/3     Running   0          4m39s
vault-0                              1/1     Running   0          24h
vault-csi-provider-c65gm             1/1     Running   0          14m
```

# Define a SecretProviderClass resource

Take a look at the file called `secretProviderClassVaultSchoolApp.yaml`

This `SecretProviderClass` will get created as part of the helm chart in the next section.

## Redeploy the API Helm Chart

Let's re-install the API with Vault

```bash
helm upgrade api --namespace schoolapp --set vault.status=kv_static_csi,vault.kind=csi schoolapp/schoolapp-api
# Port forward the api
kubectl port-forward service/api 5000:5000
# Port forward the frontend
kubectl port-forward service/frontend 8001:8080
```

Take a closer look at the spec in [the API deployment template of the School App](https://gitlab.com/public-projects3/training/school-app/-/blob/main/deployments/chart-api/templates/api-deployment.yaml)

```yaml
...
    spec:
      containers:
        - image: {{ .Values.api.image.repository }}:latest
          imagePullPolicy: {{ .Values.api.image.pullPolicy }}
          {{ if eq .Values.vault.kind "csi" }}
          volumeMounts:
            - name: 'vault-db-creds'
              mountPath: '/app/secrets/'
              readOnly: true
          {{ end }}
      {{ if eq .Values.vault.kind "csi" }}
      volumes:
        - name: vault-db-creds
          csi:
            driver: 'secrets-store.csi.k8s.io'
            readOnly: true
            volumeAttributes:
              secretProviderClass: 'schoolapp'
      {{ end }}
...
```

## Check the MongoDB Creds Secrets inside the API Container

Exec into the API pod and take a look at the `schoolapp-mongodb-username` and `schoolapp-mongodb-password` files that were inserted inside of the folder `/app/secrets`

```bash
kubectl exec -it $(kubectl get pods --template '{{range .items}}{{.metadata.name}}{{end}}' --selector=app=api) -- bash
ls /app/secrets
```

**Details**
```
root@api-5888d58476-ngmdr:/app/app# cd /app/secrets
root@api-5888d58476-ngmdr:/app/secrets# ls
schoolapp-mongodb-password  schoolapp-mongodb-username
root@api-5888d58476-ngmdr:/app/secrets# cat schoolapp-mongodb-username
schoolapp
root@api-5888d58476-ngmdr:/app/secrets# cat schoolapp-mongodb-password
mongoRootPass
```

> This concludes this lab