# Vault Agent Template for Vault Unaware Apps

In this lab we will re-run the School App but instead of using the Vault token to talk to the Vault API to get the MongoDB credentials (Vault aware app), we'll use the vault agent template to drop the MongoDB credentials into a file where our app will use to log into the DB (Vault unaware app).

## Redeploy the API Helm Chart

Let's re-install the API with Vault

```bash
helm upgrade api --namespace schoolapp --set vault.status=kv_static_injector_template_file,vault.kind=injector schoolapp/schoolapp-api
# Port forward the api
kubectl port-forward service/api 5000:5000
# Port forward the frontend
kubectl port-forward service/frontend 8001:8080
```

Take a closer look at the annotations in [the API deployment template of the School App](https://gitlab.com/public-projects3/training/school-app/-/blob/main/deployments/chart-api/templates/api-deployment.yaml)

Pay attention to the template section.

```yaml
  template:
    metadata:
    {{ if eq .Values.vault.kind "injector" }}
      annotations:
        vault.hashicorp.com/agent-inject: "true"
        vault.hashicorp.com/agent-inject-token: "true"
        vault.hashicorp.com/agent-inject-status: "update"
        vault.hashicorp.com/role: {{ .Values.serviceAccount.name | quote }}
        vault.hashicorp.com/secret-volume-path: "/app/secrets/"
      {{ if eq .Values.vault.status "kv_static_injector_template_file" }}
        vault.hashicorp.com/agent-inject-secret-schoolapp-mongodb-username: "internal/data/schoolapp/mongodb"
        vault.hashicorp.com/agent-inject-secret-schoolapp-mongodb-password: "internal/data/schoolapp/mongodb"
        # Notice the use of {{` {{some GO templating}} `}} the backticks and {{``}} is used to escape the templating so that helm doesn't error out when it tries to render the templating when it shouldn't. So it treats it as a literal string.
        vault.hashicorp.com/agent-inject-template-schoolapp-mongodb-username: |
          {{`{{- with secret "internal/data/schoolapp/mongodb" -}}
          {{ .Data.data.schoolapp_DB_USERNAME }}
          {{- end -}}
        vault.hashicorp.com/agent-inject-template-schoolapp-mongodb-password: |
          {{- with secret "internal/data/schoolapp/mongodb" -}}
          {{ .Data.data.schoolapp_DB_PASSWORD }}
          {{- end -}}`}}
      {{ end }}
    {{ end }}
```

We are using the `schoolapp-mongodb-username` file for the username and the `schoolapp-mongodb-password` file for the password. Both files are dropped into the `/app/secrets/` folder.


## Check the School App

Run some actions on the School App and check the logs of the API container.

```bash
kubectl logs -n schoolapp -f $(kubectl get pods --template '{{range .items}}{{.metadata.name}}{{end}}' --selector=app=api) -c api
```

Notice this output:

**Expected Output**
```
MongoDB Credentials Using Vault KV with injector and templates for Vault unaware apps: 
Username = schoolapp
Password = mongoRootPass
```

## Check the Vault Secrets inside the API Container

Exec into the API pod and take a look at the token that was inserted inside of the file `/app/secrets/token`

```bash
kubectl exec -it -n schoolapp $(kubectl get pods --template '{{range .items}}{{.metadata.name}}{{end}}' --selector=app=api) -- bash
cat cat /app/secrets/schoolapp-mongodb-username
cat cat /app/secrets/schoolapp-mongodb-password
```

**Expected Output:**
```
cat /app/secrets/schoolapp-mongodb-username
schoolapp
cat /app/secrets/schoolapp-mongodb-password
mongoRootPass
```

As you can see the app's file system has the secrets dropped in by Vault. The app doesn't talk to Vault directly. All it needs to know is what files to find the secrets in.

> This concludes this lab