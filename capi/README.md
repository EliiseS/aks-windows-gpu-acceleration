# Create a CAPZ cluster w/ GPU enabled Windows Node

## Install clusterctl tool

```bash
curl -L https://github.com/kubernetes-sigs/cluster-api/releases/download/v1.3.3/clusterctl-linux-amd64 -o clusterctl
sudo install -o root -g root -m 0755 clusterctl /usr/local/bin/clusterctl
clusterctl version
```
## Create management cluster

```bash
# Ex: kind
kind create cluster --name capi
```

## Create Service Principal

```bash
export AZURE_SUBSCRIPTION_ID="<SubscriptionId>"
az ad sp create-for-rbac --role contributor --scopes="/subscriptions/${AZURE_SUBSCRIPTION_ID}"
```

## Set env vars

```bash
export EXP_MACHINE_POOL=true

export AZURE_TENANT_ID="<Tenant>"
export AZURE_CLIENT_ID="<AppId>"
export AZURE_CLIENT_SECRET="<Password>"

export AZURE_SUBSCRIPTION_ID_B64="$(echo -n "$AZURE_SUBSCRIPTION_ID" | base64 | tr -d '\n')"
export AZURE_TENANT_ID_B64="$(echo -n "$AZURE_TENANT_ID" | base64 | tr -d '\n')"
export AZURE_CLIENT_ID_B64="$(echo -n "$AZURE_CLIENT_ID" | base64 | tr -d '\n')"
export AZURE_CLIENT_SECRET_B64="$(echo -n "$AZURE_CLIENT_SECRET" | base64 | tr -d '\n')"

export AZURE_CLUSTER_IDENTITY_SECRET_NAME="cluster-identity-secret"
export CLUSTER_IDENTITY_NAME="cluster-identity"
export AZURE_CLUSTER_IDENTITY_SECRET_NAMESPACE="default"
```

## Prepare management cluster

```bash
kubectl create secret generic "${AZURE_CLUSTER_IDENTITY_SECRET_NAME}" --from-literal=clientSecret="${AZURE_CLIENT_SECRET}" --namespace "${AZURE_CLUSTER_IDENTITY_SECRET_NAMESPACE}"

clusterctl init --infrastructure azure
```

## Create cluster

```bash

# !! Replace "${SSH_PUB_KEY_B64_ENCODED}" and "${SSH_PUB_KEY}" with your ssh pub key encoded and plain text respectively !!
# You may also need to update the VM sizes based on availability
kubectl apply -f machinepool-win-gpu-cluster.yaml
```

## Install CNI

```bash
export IPV4_CIDR_BLOCK=$(kubectl get cluster win-gpu -o=jsonpath='{.spec.clusterNetwork.pods.cidrBlocks[0]}')

clusterctl get kubeconfig win-gpu > ~/.kube/win-gpu.kubeconfig
export KUBECONFIG=~/.kube/win-gpu.kubeconfig

helm repo add projectcalico https://docs.tigera.io/calico/charts && \
helm install calico projectcalico/tigera-operator -f https://raw.githubusercontent.com/kubernetes-sigs/cluster-api-provider-azure/main/templates/addons/calico/values.yaml --set-string "installation.calicoNetwork.ipPools[0].cidr=${IPV4_CIDR_BLOCK}" --namespace tigera-operator --create-namespace

# Windows node
kubectl get configmap kubeadm-config --namespace=kube-system -o yaml \
| sed 's/namespace: kube-system/namespace: calico-system/' \
| kubectl create -f -

kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/cluster-api-provider-azure/f304a5532c53ef4483c2ede10b0532ae5c63ec9d/templates/addons/windows/calico/calico.yaml
```