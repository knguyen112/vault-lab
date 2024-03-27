# School App Example with ArgoCD and the Vault CSI Provider

In this lab we will see how to use a GitOps approach with ArgoCD to run our school app with the Vault CSI provider.

## Test the School App with the Vault CSI Provider

### Redeploy the API Helm Chart for Vault

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
NAME                           READY   STATUS    RESTARTS      AGE
pod/vault-0                    1/1     Running   2 (22m ago)   2d17h
pod/vault-csi-provider-v4xb2   1/1     Running   0             22s
```

### Install the secrets store CSI driver

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
NAME                                 READY   STATUS    RESTARTS      AGE
csi-secrets-store-csi-driver-fvmp4   3/3     Running   0             39s
vault-0                              1/1     Running   2 (24m ago)   2d17h
vault-csi-provider-v4xb2             1/1     Running   0             2m15s
```

### Update the Values File

Update the `values.yaml` file under `schoolapp-subchart` with these values:

```yaml
schoolapp-api:
  vault:
    status: 'kv_static_csi'
    kind: 'csi'
```

Commit and push the changes to GitLab. Then hit refresh in the argoCD UI.

Notice the app is now out of sync.

Click the APP DIFF button to see the changes and make sure the changes make sense then Click the sync button and Synchronize

Now run

```bash
kubectl get po
```

**Expected Output:**

```
NAME                                 READY   STATUS    RESTARTS   AGE
api-7b7b4c9588-qr9cv                 1/1     Running   0          2m29s
frontend-6c445b8775-jfxgv            1/1     Running   0          2m29s
schoolapp-mongodb-6cdf54d797-crs9f   1/1     Running   0          2m29s
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
MongoDB Credentials Using Vault KV and the Vault CSI Provider for Vault unaware apps: 
Username = schoolapp
Password = mongoRootPass
```

> This concludes this lab