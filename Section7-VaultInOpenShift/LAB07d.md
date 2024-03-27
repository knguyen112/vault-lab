# Deploy the School App with ArgoCD and Vault Injector

In this lab we will deploy the schoolapp and use the vault agent injector to provide the mongoDB credentials.

## Delete the Existing School App

```bash
oc delete project schoolapp
```

## Create an ArgoCD Application

Note that we need a label on the schoolapp namespace to indicate that this namespace is managed by ArgoCD as shown below:

```bash
oc new-project schoolapp
oc label namespaces schoolapp argocd.argoproj.io/managed-by=argocd
cat >argocdSchoolApp.yaml <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: schoolapp
  namespace: argocd
spec:
  destination:
    namespace: schoolapp
    server: 'https://kubernetes.default.svc'
  source:
    path: Section7-VaultInOpenShift/schoolapp-subchart
    repoURL: 'https://gitlab.com/public-projects3/training/vault-201.git'
    targetRevision: HEAD
  project: default
  syncPolicy:
    syncOptions:
      - CreateNamespace=true
EOF
oc apply -f argocdSchoolApp.yaml
```

We're using the vault injector in this case and the app should work fine. Go to the URL: http://schoolapp-schoolapp.apps-crc.testing to see the app and interact with it.

The URL above should work because your local hosts file on your computer should contain the hostname.

## Check the Logs

You can check the API logs as well:

```bash
oc logs -n schoolapp -f $(oc get pods --template '{{range .items}}{{.metadata.name}}{{end}}' --selector=app=api) -c api
```

Notice this output:

**Expected Output**
```
MongoDB Credentials Using Vault KV with injector and templates for Vault unaware apps: 
Username = schoolapp
Password = mongoRootPass
```

> This concludes this lab