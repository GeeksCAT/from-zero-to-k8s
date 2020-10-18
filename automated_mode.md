# k3sup

k3sup (aka ketchup) utility to create k3s clusters in seconds :)

```bash
curl -sLS https://get.k3sup.dev | sh
sudo install k3sup /usr/local/bin/
```

## Create cluster

### Server

Install everything:
```bash
export SERVER_IP=XXX.XXX.XXX.XXX
k3sup install --ip $SERVER_IP --user root --local-path=./kubeconfig
```

Use this kubeconfig and validate cluster:
```bash
export KUBECONFIG=`pwd`/kubeconfig
kubectl get nodes
```

### Workers

Install everything:
```bash
export IP=XXX.XXX.XXX.XXX
k3sup join --ip $IP --server-ip $SERVER_IP --user root
```

Confirm that worker belongs to our cluster:
```bash
kubectl get nodes
```

## Deploy kubernetes-dashboard

Deploy it with:
```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.4/aio/deploy/recommended.yaml
```

### Create Service Account

https://github.com/kubernetes/dashboard/blob/master/docs/user/access-control/creating-sample-user.md

Create account:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
EOF
```

Bind to ClusterRole cluster-admin

```bash
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard
EOF
```

Extract token for login

```bash
kubectl -n kubernetes-dashboard describe secret $(kubectl -n kubernetes-dashboard get secret | grep admin-user | awk '{print $1}')
```

### Access the dashboard

```bash
kubectl proxy
```

Open http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/#/login in a browser

Login with the previously extracted token

Enjoy :)


## Helm

Install helm locally from GH or your package system

Activate official charts

```bash
helm repo add stable https://kubernetes-charts.storage.googleapis.com
helm repo update
```


## Install cert-manager

```bash
kubectl create namespace cert-manager

helm repo add jetstack https://charts.jetstack.io && helm repo update
helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --version v1.0.3 \
  --set installCRDs=true

# workaround for 1.0.3
# kubectl delete mutatingwebhookconfiguration.admissionregistration.k8s.io cert-manager-webhook
# kubectl delete validatingwebhookconfigurations.admissionregistration.k8s.io cert-manager-webhook
```

### Prepare cert issuers

#### Staging

```bash
cat <<EOF | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    email: it@geekscat.org
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-staging
    solvers:
    - http01: {}
EOF
```

#### Production

```bash
cat <<EOF | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    email: it@geekscat.org
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01: {}
EOF
```

### HackMD

```bash
cat <<EOF | cat hackmd-values.yaml
ingress:
  enabled: true
  annotations:
    {}
  path: /
  hosts:
    - hackmd.nemperfeina.cat
  tls: []
  #  - secretName: hackmd-nemperfeina-cat-tls
  #    hosts:
  #      - hackmd.nemperfeina.cat

resources:
  {}
  # limits:
  #  cpu: 100m
  #  memory: 128Mi
  # requests:
  #  cpu: 100m
  #  memory: 128Mi

persistence:
  enabled: true
  ## hackmd data Persistent Volume access modes
  ## Must match those of existing PV or dynamic provisioner
  ## Ref: http://kubernetes.io/docs/user-guide/persistent-volumes/
  ##
  accessModes:
    - ReadWriteOnce
  annotations: {}
  existingClaim: ''
  size: 2Gi
  ## A manually managed Persistent Volume and Claim
  ## Requires persistence.enabled: true
  ## If defined, PVC must be created manually before volume will be bound
  # existingClaim:

  ## database data Persistent Volume Storage Class
  ## If defined, storageClassName: <storageClass>
  ## If set to "-", storageClassName: "", which disables dynamic provisioning
  ## If undefined (the default) or set to null, no storageClassName spec is
  ##   set, choosing the default provisioner.  (gp2 on AWS, standard on
  ##   GKE, AWS & OpenStack)
  ##
  storageClass: local-path

podAnnotations: {}

extraVars:
  - name: HMD_IMAGE_UPLOAD_TYPE
    value: filesystem
  - name: CMD_DOMAIN
    value: hackmd.nemperfeina.cat
  - name: CMD_PROTOCOL_USESSL
    value: 'true'
  - name: CMD_URL_ADDPORT
    value: 'false'
# postgresql:
#   storageClass: local-path
#   persistence:
#     storageClass: local-path
#   global:
#     storageClass: local-path

# global:
#   storageClass: local-path
#   persistence:
#     storageClass: local-path
EOF
```

And install it with helm:
```
kubectl create namespace hackmd
helm install hackmd --namespace hackmd -f hackmd-values.yaml stable/hackmd
```
