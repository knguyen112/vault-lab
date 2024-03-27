# School App Example with ArgoCD and the Vault Injector

In this lab we will see how to use a GitOps approach with ArgoCD to run our school app while using the Vault injector for secrets.

## Redeploy the API Helm Chart for Vault

Let's re-install Vault with enabling the the injector and disabling CSI

From the Vault namespace run the following:

```bash
helm upgrade vault --namespace vault \
    --set server.image.tag=1.9.3 \
    --set injector.enabled=true \
    --set csi.enabled=false \
    hashicorp/vault
```

## Test the School App with the Vault Injector

Update the `values.yaml` file under `schoolapp-subchart` with these values:

```yaml
schoolapp-api:
  vault:
    status: 'kv_static_injector_template_file'
    kind: 'injector'
```

Here we're using templating for a Vault unaware app similar to what we did in Lab04b 

Commit and push the changes to GitLab. Then hit refresh in the argoCD UI.

Notice the app is now out of sync.

Click the APP DIFF button to see the changes and make sure the changes make sense then Click the sync button and Synchronize

Notice how the api pod now has 2/2 containers because the vault agent container is now running beside the api container.

**Expected Output:**

```
NAME                                     READY   STATUS    RESTARTS      AGE
pod/api-657f89d85b-ssnld                 2/2     Running   0             28s
pod/frontend-6c445b8775-ttgdx            1/1     Running   2 (14m ago)   52m
pod/schoolapp-mongodb-6cdf54d797-s2tnr   1/1     Running   4 (14m ago)   52m
```

Now let's check the school app

Make sure the api and frontend ports are being exposed to the host machine:

```bash
# Port forward the frontend
kubectl port-forward service/frontend 8001:8080 -n schoolapp
# Port forward the api
kubectl port-forward service/api 5000:5000 -n schoolapp
```

Try creating and deleting a course and check the logs of the api pod.

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

> This concludes this lab
