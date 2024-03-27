# School App Example with the Vault Agent Sidecar Injector Lab

In this lab we will re-run the School App but instead of hardcoding the mongoDB credentials, we'll leverage Vault via the Vault Agent Injector.

## Redeploy the API Helm Chart

Let's re-install the API with Vault

```bash
helm upgrade api --namespace schoolapp --set vault.status=kv_static,vault.kind=injector schoolapp/schoolapp-api
# Port forward the api
kubectl port-forward service/api 5000:5000
# Port forward the frontend
kubectl port-forward service/frontend 8001:8080
```

Take a closer look at the annotations in [the API deployment template of the School App](https://gitlab.com/public-projects3/training/school-app/-/blob/main/deployments/chart-api/templates/api-deployment.yaml)

```yaml
annotations:
    vault.hashicorp.com/agent-inject: "true"
    vault.hashicorp.com/agent-inject-token: "true"
    vault.hashicorp.com/agent-inject-status: "update"
    vault.hashicorp.com/role: {{ .Values.serviceAccount.name | quote }}
    vault.hashicorp.com/secret-volume-path: "/app/secrets/"
```

## Check the School App

Run some actions on the School App and check the logs of the API container.

In a browser go to http://127.0.0.1:8001 

```bash
kubectl logs -f $(kubectl get pods --template '{{range .items}}{{.metadata.name}}{{end}}' --selector=app=api) -c api
```

Notice the `Vault token` output.

## Check the Vault Token inside the API Container

Exec into the API pod and take a look at the token that was inserted inside of the file `/app/secrets/token`

```bash
kubectl exec -it $(kubectl get pods --template '{{range .items}}{{.metadata.name}}{{end}}' --selector=app=api) -- bash
cat /app/secrets/token
```

Notice how the token keeps changing. Remember that we configured the `ttl=5s` and `max_ttl=20s` in the Kubernetes role:

```bash
vault write auth/kubernetes/role/schoolapp \
    bound_service_account_names=schoolapp \
    bound_service_account_namespaces=schoolapp \
    policies=schoolapp \
    ttl=5s \
    max_ttl=20s
```

So the Vault agent sidecar will automatically renew the token for us every 5s up to a max of 20s. After 20s, the agent will create a new token and drop it in the file `/app/secrets/token` where the application will pick it up to authenticate into Vault. In this case, the app doesn't need to because due to the static nature of the MongoDB creds, they are stored in the apps memory. When we start using dynamic DB creds, we need to add logic to the application to re-read the creds from Vault and re-authenticate the API to mongoDB.

> This concludes this lab