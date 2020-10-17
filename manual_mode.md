# Prepare the cluster

## Init all nodes

```bash
apt update && apt upgrade -y
apt autoremove
reboot
```

## Install k3s

### master

```bash
ip addr show
curl -sfL https://get.k3s.io | sh -
systemctl status k3s

# extract token
cat /var/lib/rancher/k3s/server/node-token

# extract kubeconfig
cat /etc/rancher/k3s/k3s.yaml
```

### local

Save master `kubeconfig` in local as `~/.kube/config` replacing `server: https://127.0.0.1:6443` with the public IP of our master

Now our local instance points to our cluster

```bash
kubectl get nodes

NAME   STATUS   ROLES    AGE     VERSION
k8s1   Ready    master   7m26s   v1.18.9+k3s1
```

### workers

Export K3S_TOKEN with the value of our master `token`

```bash
export K3S_TOKEN=K103e66b7319e733d5efd332e315d2d7c840f9435b3f76018f1a47402ec4d17b9c9::server:72f8b7e9cab0c18eb3d844c96f9c7c83
export K3S_MASTER_IP_PRIVATE=10.20.10.6
curl -sfL https://get.k3s.io | K3S_URL=https://$K3S_MASTER_IP_PRIVATE:6443 K3S_TOKEN=$K3S_TOKEN sh -
```

Validate that worker has been added to our cluster :)

```bash
kubectl get nodes
NAME   STATUS   ROLES    AGE   VERSION
k8s1   Ready    master   15m   v1.18.9+k3s1
k8s2   Ready    <none>   7s    v1.18.9+k3s1
```

Since this step, everything will be done in our local machine :)

## Prepare helm

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

kubectl delete mutatingwebhookconfiguration.admissionregistration.k8s.io cert-manager-webhook
kubectl delete validatingwebhookconfigurations.admissionregistration.k8s.io cert-manager-webhook
```

Prepare cert issuers

Staging:

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

Production:

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

## Prepare k8s-dashboard

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.4/aio/deploy/recommended.yaml
```

## Install some tools


### HackMD

kubectl create namespace hackmd
helm install hackmd --namespace hackmd -f hackmd-values.yaml stable/hackmd


### Docker Registry

kubectl create namespace docker-registry
helm install --name docker-registry --namespace docker-registry -f registry-values.yaml stable/docker-registry
