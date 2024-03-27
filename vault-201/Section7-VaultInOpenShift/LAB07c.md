# Use the ArgoCD Vault Plugin

In this lab we will use another method to retrieve secrets from Vault using the [ArgoCD Vault plugin.](https://argocd-vault-plugin.readthedocs.io/en/stable/)

## Build Your Docker Image for the Plugin

You can use my image `samgabrail/argocdvaultplugin` that I built that utilizes v1.10.1 of the plugin or go ahead and create your own using the Dockerfile in this folder `Section7-VaultInOpenShift`. You can change the `AVP_VERSION` variable to the version of the plugin you desire.

## Create a new ArgoCD Instance with the Plugin

You may need to bounce the CRC environment from time to time, run the following:

```bash
crc stop
crc start
```

We will need to create a new ArgoCD instance that uses the ArgoCD Vault plugin.

```bash
oc new-project argocd
cat >vpluginServiceAccount.yaml <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: vplugin
  namespace: argocd
EOF
oc apply -f vpluginServiceAccount.yaml
cat >argocdInstance.yaml <<EOF
apiVersion: argoproj.io/v1alpha1
kind: ArgoCD
metadata:
  name: argocd
  namespace: argocd
spec:
  server:
    autoscale:
      enabled: false
    grpc:
      ingress:
        enabled: false
    ingress:
      enabled: false
    route:
      enabled: true
    service:
      type: ''
  grafana:
    enabled: false
    ingress:
      enabled: false
    route:
      enabled: false
  prometheus:
    enabled: false
    ingress:
      enabled: false
    route:
      enabled: false
  initialSSHKnownHosts: {}
  rbac: {}
  repo:
    image: samgabrail/argocdvaultplugin
    mountsatoken: true
    serviceaccount: vplugin
    version: latest
  dex:
    openShiftOAuth: false
  version: latest
  ha:
    enabled: false
  tls:
    ca: {}
  image: samgabrail/argocdvaultplugin
  redis:
  configManagementPlugins: |-
    - name: argocd-vault-plugin
      generate:
        command: ["argocd-vault-plugin"]
        args: ["generate", "./"]
  controller:
    processors: {}
EOF
oc apply -f argocdInstance.yaml
```

Run the following command to make sure everything is up
```bash
oc get all
```

Open a new browser window and go to the link mentioned in the route section:

**Expected Output**
```
NAME                                      READY   STATUS    RESTARTS   AGE
pod/argocd-application-controller-0       1/1     Running   0          7m25s
pod/argocd-dex-server-7f7779698b-kzdxs    1/1     Running   0          7m25s
pod/argocd-redis-ccc9b98d7-lnz2f          1/1     Running   0          7m26s
pod/argocd-repo-server-754c98cc57-pqw7j   1/1     Running   0          28s
pod/argocd-server-79466c6854-g9xcr        1/1     Running   0          7m26s

NAME                            TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)             AGE
service/argocd-dex-server       ClusterIP   10.217.4.2     <none>        5556/TCP,5557/TCP   7m27s
service/argocd-metrics          ClusterIP   10.217.5.125   <none>        8082/TCP            7m27s
service/argocd-redis            ClusterIP   10.217.5.204   <none>        6379/TCP            7m27s
service/argocd-repo-server      ClusterIP   10.217.4.76    <none>        8081/TCP,8084/TCP   7m27s
service/argocd-server           ClusterIP   10.217.4.107   <none>        80/TCP,443/TCP      7m27s
service/argocd-server-metrics   ClusterIP   10.217.5.166   <none>        8083/TCP            7m27s

NAME                                 READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/argocd-dex-server    1/1     1            1           7m27s
deployment.apps/argocd-redis         1/1     1            1           7m26s
deployment.apps/argocd-repo-server   1/1     1            1           7m26s
deployment.apps/argocd-server        1/1     1            1           7m26s

NAME                                            DESIRED   CURRENT   READY   AGE
replicaset.apps/argocd-dex-server-67f9b7cdd4    0         0         0       7m26s
replicaset.apps/argocd-dex-server-7f7779698b    1         1         1       7m25s
replicaset.apps/argocd-redis-ccc9b98d7          1         1         1       7m26s
replicaset.apps/argocd-repo-server-754c98cc57   1         1         1       28s
replicaset.apps/argocd-server-79466c6854        1         1         1       7m26s

NAME                                             READY   AGE
statefulset.apps/argocd-application-controller   1/1     7m26s

NAME                                     HOST/PORT                               PATH   SERVICES        PORT    TERMINATION            WILDCARD
route.route.openshift.io/argocd-server   argocd-server-argocd.apps-crc.testing          argocd-server   https   passthrough/Redirect   None
```

### Test that the Plugin is present

Exec into the container:

```bash
oc exec -it argocd-repo-server-754c98cc57-ztl59 -- bash
ls -lah /usr/local/bin
```

**Expected Output:**
```
...
-rwxr-xr-x. 1 root root  64M Mar 29 19:16 argocd-vault-plugin
...
```


### Get Route

In this case navigate to https://argocd-server-argocd.apps-crc.testing. Your local computer's host file should already have the proper DNS setup.

### Get Admin Password

To log into ArgoCD, use the username `admin` and get the password from the `argocd-cluster` as shown below:
```bash
oc -n argocd get secret argocd-cluster -o jsonpath='{.data.admin\.password}' | base64 -d
```

## Update Vault Configuration

```bash
oc project vault
oc exec -it pod/vault-0 -- sh
export VAULT_ADDR=http://127.0.0.1:8200
vault login root
vault policy write vplugin - <<EOF
path "secret/data/vplugin/supersecret" {
  capabilities = ["read"]
}
path "internal/data/schoolapp/mongodb" {
  capabilities = ["list","read"]
}
EOF
vault write auth/kubernetes/role/vplugin \
    bound_service_account_names=vplugin \
    bound_service_account_namespaces=argocd \
    policies=vplugin \
    ttl=24h
```

Type exit to exit the vault container execution and back to the Azure VM prompt.

## Create the School App Project

```bash
oc new-project schoolapp
oc label namespaces schoolapp argocd.argoproj.io/managed-by=argocd
```

Note that we need a label on the schoolapp namespace to indicate that this namespace is managed by ArgoCD as shown in the command above.

## Create a Role and a RoleBinding for ArgoCD

We need this to allow ArgoCD to create resources in the schoolapp namespace

Create the role:

```bash
cat << EOF >> argocdschoolappRole.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: argocdschoolapp
  namespace: schoolapp
rules:
- apiGroups:
  - "*"
  resources:
  - "*"
  verbs:
  - Get
  - List
  - Watch
  - Patch
EOF
oc apply -f  argocdschoolappRole.yaml
```

Create the RoleBinding:

```bash
cat << EOF >> argocdschoolappRoleBinding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: argocdschoolapp
  namespace: schoolapp
roleRef:
  apiGroup: rbac.authorization.k8s.io  
  kind: Role
  name: argocdschoolapp
subjects:
- kind: ServiceAccount
  name: argocd-application-controller-0 
  namespace: argocd
EOF
oc create -f  argocdschoolappRoleBinding.yaml
```

## Build an Argo App with the Plugin

First delete the current School App then run the following:

```bash
cat >argocdSchoolAppWithPlugin.yaml <<EOF
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
    path: Section7-VaultInOpenShift/schoolappWithArgoCDvaultPlugin
    repoURL: 'https://gitlab.com/public-projects3/training/vault-201.git'
    targetRevision: HEAD
    plugin:
      name: argocd-vault-plugin
      env:
        - name: AVP_K8S_ROLE
          value: vplugin
        - name: AVP_TYPE
          value: vault
        - name: VAULT_ADDR
          value: http://vault.vault.svc.cluster.local:8200
        - name: AVP_AUTH_TYPE
          value: k8s
  project: default
  syncPolicy:
    syncOptions:
      - CreateNamespace=true
EOF
oc apply -f argocdSchoolAppWithPlugin.yaml
```

## Check the Secret

In the OpenShift UI, go to the `schoolapp` project and `secrets` and check the content of the `schoolapp` secret to see the username and password for MongoDB. Basically, the ArgoCD Vault Plugin grabbed the MongoDB creds from Vault and populated a K8s secret with them.

> This concludes this lab