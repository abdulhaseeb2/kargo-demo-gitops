# Instructions

Distributed Kargo lab on two kind clusters on your cloned GitOps repo

## Prerequisites

### Tools

- [kind](https://kind.sigs.k8s.io/)
- [kubectl](https://kubernetes.io/docs/reference/kubectl/)
- [Helm v3.13.1+](https://helm.sh/docs/intro/install/)
- htpasswd (from apache2-utils or httpd-tools)
- openssl
- [git](https://git-scm.com/)

### GitOps repository layout

Your gitops repository should have the following structure with the mentioned files

> - bootstrap/argocd/main-demo-app.yaml
> - bootstrap/argocd/shard-demo-app.yaml
> - k8s/main-app/namespace.yaml
> - k8s/main-app/deployment.yaml
> - k8s/main-app/service.yaml
> - k8s/main-app/kustomization.yaml
> - k8s/shard-app/namespace.yaml
> - k8s/shard-app/deployment.yaml
> - k8s/shard-app/service.yaml
> - k8s/shard-app/kustomization.yaml
> - kargo/project.yaml
> - kargo/warehouse-main.yaml
> - kargo/warehouse-shard.yaml
> - kargo/stage-main.yaml
> - kargo/stage-shard.yaml

## Clone Repo

```bash
git clone https://github.com/<your-org>/<your-repo>.git
cd kargo-demo-gitops
```

## Environment variables

```bash
export GITOPS_REPO_URL="https://github.com/<your-org>/<your-repo>.git"
export GITHUB_USERNAME="<your-username>"
export GITHUB_PAT="<your-personal-access-token>"
```

## Create main and shard clusters

```bash
kind create cluster --config /manifests/kind-main.yaml
kind create cluster --config /manifests/kind-shard.yaml
```

Verify

```bash
kubectl --context kind-kargo-main get nodes
kubectl --context kind-kargo-shard get nodes
```

## Deploy Argo CD on both clusters

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

kubectl config use-context kind-kargo-main 
kubectl --context kind-kargo-main create ns argocd
helm upgrade --install argocd argo/argo-cd \
  --namespace argocd \
  --kube-context kind-kargo-main \
  --wait

kubectl config use-context kind-kargo-shard 
kubectl --context kind-kargo-shard create ns argocd
helm upgrade --install argocd argo/argo-cd \
  --namespace argocd \
  --kube-context kind-kargo-shard \
  --wait
```

Verify

```bash
kubectl --context kind-kargo-main -n argocd get pods
kubectl --context kind-kargo-shard -n argocd get pods
```

## Create Argo CD repository secrets

```bash
for ctx in kind-kargo-main kind-kargo-shard; do
  kubectl --context "$ctx" -n argocd create secret generic repo-kargo-demo-gitops \
    --type Opaque \
    --from-literal=type=git \
    --from-literal=url="${GITOPS_REPO_URL}" \
    --from-literal=username="${GITHUB_USERNAME}" \
    --from-literal=password="${GITHUB_PAT}" \
    --dry-run=client -o yaml | kubectl --context "$ctx" -n argocd apply -f -

  kubectl --context "$ctx" -n argocd label secret repo-kargo-demo-gitops \
    argocd.argoproj.io/secret-type=repository --overwrite
done
```

## Install cert-manager and Kargo on the main cluster

Install cert-manager

```bash
helm install cert-manager oci://quay.io/jetstack/charts/cert-manager \
  --version v1.20.2 \
  --namespace cert-manager \
  --create-namespace \
  --kube-context kind-kargo-main \
  --set crds.enabled=true \
  --wait
```

Generate and save the Kargo UI password

```bash
kubectl config use-context kind-kargo-main

pass=$(openssl rand -base64 48 | tr -d "=+/" | head -c 32)
hashed_pass=$(htpasswd -bnBC 10 "" "$pass" | tr -d ':\n')
signing_key=$(openssl rand -base64 48 | tr -d "=+/" | head -c 32)
echo "Kargo admin password: $pass"
```

Generate kargo values file for main cluster

```bash
cat > kargo-main-values.yaml <<EOF
controller:
  enabled: true
  rollouts:
    integrationEnabled: false
  argocd:
    integrationEnabled: true
    namespace: argocd
    watchArgocdNamespaceOnly: true

api:
  adminAccount:
    passwordHash: ${hashed_pass}
    tokenSigningKey: ${signing_key}
  argocd:
    urls:
      "": "https://localhost:30080"
      "shard-a": "https://localhost:31080"
EOF
```

Install Kargo

```bash
helm upgrade --install kargo oci://ghcr.io/akuity/kargo-charts/kargo \
  --namespace kargo \
  --create-namespace \
  --kube-context kind-kargo-main \
  -f kargo-main-values.yaml \
  --wait
```

Verify

```bash
kubectl --context kind-kargo-main -n kargo get pods
```

## Create remote API access for the shard controller

> Important: Do not use kind get kubeconfig directly. It points to 127.0.0.1, which is unreachable from inside shard controller pods.

### Create service account on main cluster

For the sake of demo we are giving the shard-controller cluster-admin access, for production or any other environments please associate relevant RBAC

```bash
kubectl --context kind-kargo-main -n kargo create sa shard-controller-remote \
  --dry-run=client -o yaml | kubectl apply -f -

kubectl --context kind-kargo-main create clusterrolebinding shard-controller-remote-admin \
  --clusterrole=cluster-admin \
  --serviceaccount=kargo:shard-controller-remote \
  --dry-run=client -o yaml | kubectl apply -f -
```

### Build reachable kubeconfig and store in shard cluster

```bash
kubectl config use-context kind-kargo-main

TOKEN=$(kubectl --context kind-kargo-main -n kargo create token shard-controller-remote --duration=24h)
MAIN_PORT=$(docker inspect kargo-main-control-plane \
  --format '{{(index (index .NetworkSettings.Ports "6443/tcp") 0).HostPort}}')

cat > kargo-main-remote.kubeconfig <<EOF
apiVersion: v1
kind: Config
clusters:
- name: kargo-main
  cluster:
    server: https://host.docker.internal:${MAIN_PORT}
    insecure-skip-tls-verify: true
users:
- name: shard-controller
  user:
    token: ${TOKEN}
contexts:
- name: shard-to-main
  context:
    cluster: kargo-main
    user: shard-controller
current-context: shard-to-main
EOF

kubectl --context kind-kargo-shard create ns kargo
kubectl --context kind-kargo-shard -n kargo create secret generic kargo-controlplane-kubeconfig \
  --from-file=kubeconfig.yaml=kargo-main-remote.kubeconfig \
  --dry-run=client -o yaml | kubectl --context kind-kargo-shard -n kargo apply -f -
```

## Install the shard kargo controller

Create the values file

```bash
cat > kargo-shard-values.yaml <<'EOF'
api:
  enabled: false
webhooksServer:
  enabled: false
externalWebhooksServer:
  enabled: false
managementController:
  enabled: false
garbageCollector:
  enabled: false

controller:
  enabled: true
  id: shard-a
  shardName: shard-a
  isDefault: false
  rollouts:
    integrationEnabled: false
  argocd:
    integrationEnabled: true
    namespace: argocd
    watchArgocdNamespaceOnly: true
  kubeconfigSecrets:
    kargo: kargo-controlplane-kubeconfig

kubeconfigSecrets:
  kargo: kargo-controlplane-kubeconfig

dataPlane:
  install: false

argocd:
  dataPlane:
    install: true
EOF
```

Deploy Kargo in the shard cluster

```bash
kubectl config use-context kind-kargo-shard

helm upgrade --install kargo-shard oci://ghcr.io/akuity/kargo-charts/kargo \
  --namespace kargo \
  --create-namespace \
  --kube-context kind-kargo-shard \
  -f kargo-shard-values.yaml \
  --set-string controller.kubeconfigSecrets.kargo=kargo-controlplane-kubeconfig \
  --set-string kubeconfigSecrets.kargo=kargo-controlplane-kubeconfig \
  --wait
```

Verify

```bash
kubectl --context kind-kargo-shard -n kargo get pods
kubectl --context kind-kargo-shard -n kargo logs deploy/kargo-controller --since=5m
kubectl --context kind-kargo-main -n kargo get lease
```

## Bootstrap Argo CD applications

```bash
kubectl config use-context kind-kargo-main
kubectl --context kind-kargo-main -n argocd apply -f bootstrap/argocd/main-demo-app.yaml

kubectl config use-context kind-kargo-shard
kubectl --context kind-kargo-shard -n argocd apply -f bootstrap/argocd/shard-demo-app.yaml
```

Verify

```bash
kubectl --context kind-kargo-main -n argocd get application demo-main
kubectl --context kind-kargo-shard -n argocd get application demo-shard
```

## Apply Kargo CRs and Git credentials

Apply on main cluster only:

```bash
kubectl config use-context kind-kargo-main

kubectl apply -f kargo/project.yaml
kubectl apply -f kargo/warehouse-main.yaml
kubectl apply -f kargo/warehouse-shard.yaml
kubectl apply -f kargo/stage-main.yaml
kubectl apply -f kargo/stage-shard.yaml
```

Create Kargo Git credentials (must have kargo.akuity.io/cred-type: git):

```bash
printf '%s' "${GITHUB_USERNAME}" > /tmp/gh-user
printf '%s' "${GITHUB_PAT}" > /tmp/gh-pat

kubectl --context kind-kargo-main -n demo create secret generic gitops-repo-creds \
  --from-literal=repoURL="${GITOPS_REPO_URL%.git}" \
  --from-file=username=/tmp/gh-user \
  --from-file=password=/tmp/gh-pat \
  --dry-run=client -o yaml | kubectl apply -f -

kubectl --context kind-kargo-main -n demo label secret gitops-repo-creds \
  kargo.akuity.io/cred-type=git --overwrite

rm -f /tmp/gh-user /tmp/gh-pat
```

Refresh warehouses:

```bash
kubectl --context kind-kargo-main -n demo annotate warehouse wh-main \
  kargo.akuity.io/refresh="$(date +%s)" --overwrite
kubectl --context kind-kargo-main -n demo annotate warehouse wh-shard \
  kargo.akuity.io/refresh="$(date +%s)" --overwrite
```

## Testing

In 2 seperate terminals run the commands

```bash
kubectl -n kargo port-forward svc/kargo-api 8443:443
```

```bash
kubectl -n argocd port-forward svc/argocd-server 8080:443
```

Now you can access the [ArgoCD Dashboard](https://localhost:8080) and [Kargo Dashboard](https://localhost:8443)

### Steps

- Log in to the Kargo UI with the admin password from step 4.
- Wait until wh-main and wh-shard produce freight.
- Promote freight to stage main (main controller updates k8s/main-app).
- Promote freight to stage shard (shard controller updates k8s/shard-app).

## Validate Setup

```bash
kubectl --context kind-kargo-main -n kargo get deploy
kubectl --context kind-kargo-shard -n kargo get deploy

kubectl --context kind-kargo-main -n demo get warehouse,stage,promotions

kubectl --context kind-kargo-main -n demo-main get deploy,svc
kubectl --context kind-kargo-shard -n demo-shard get deploy,svc

kubectl --context kind-kargo-main -n demo get stage main -o jsonpath='{.spec.shard}{"\n"}{.metadata.labels.kargo\.akuity\.io/shard}{"\n"}'
kubectl --context kind-kargo-main -n demo get stage shard -o jsonpath='{.spec.shard}{"\n"}{.metadata.labels.kargo\.akuity\.io/shard}{"\n"}'
```

### Expected results

- main stage: empty shard fields
- shard stage: shard-a
- demo-main runs on main cluster only
- demo-shard runs on shard cluster only
- Git commits appear in your repo updating k8s/main-app/ and/or k8s/shard-app/

## Teardown

```bash
kind delete cluster --name kargo-main
kind delete cluster --name kargo-shard
```
