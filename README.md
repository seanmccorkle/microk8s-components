# microk8s-components

## microk8s
```BASH
snap install microk8s --classic
microk8s status
microk8s kubectl get nodes
microk8s config > microk8s-config.yaml

# Add your current user to the microk8s group
sudo usermod -a -G microk8s $USER
mkdir -p ~/.kube
chmod 0700 ~/.kube

# Refresh group memberships
newgrp microk8s

# enable community repo
microk8s enable community
microk8s addons repo list

# enable traefik and metallb
microk8s enable ingress
microk8s enable metallb

# cilium
microk8s enable cilium
microk8s cilium status
microk8s cilium hubble enable
microk8s cilium hubble enable --ui
kubectl port-forward -n kube-system svc/hubble-ui 12000:80
```

## ArgoCD

### Installation
```BASH
kubectl config current-context
kubectl create namespace argocd
kubectl apply -n argocd --server-side --force-conflicts -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl patch configmap argocd-cm -n argocd --type merge -p '{"data":{"kustomize.buildOptions":"--enable-helm"}}'
```

### CLI Installation
```BASH
curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
chmod +x argocd-linux-amd64
sudo mv argocd-linux-amd64 /usr/local/bin/argocd
argocd version --client
```

### Login
```BASH
kubectl config current-context
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
kubectl port-forward svc/argocd-server -n argocd 8080:443
argocd login localhost:8080 --insecure
argocd logout localhost:8080
```

### SSH Keys
```BASH
ssh-keygen -t ed25519 -C "argocd-github-key" -f ./id_ed25519_argocd -N ""
ssh-keygen -l -f id_ed25519_argocd
ssh-keygen -l -f id_ed25519_argocd.pub
ssh -T git@github.com -i  id_ed25519_argocd
```

**Note when configuring a GitHub repository, with SSH key, save as a credentials template**

## ApplicationSet

This repository uses an ArgoCD ApplicationSet to automatically discover and deploy applications from `apps/*/config.yaml` files.

### Supported Source Types

#### Helm Charts (External Repository)

For applications using Helm charts:

```yaml
name: my-helm-app
project: default
annotations:
  argocd.argoproj.io/sync-wave: "5"
source:
  repoURL: https://charts.example.com
  targetRevision: 1.2.3
  chart: my-chart
  helm:
    releaseName: my-release
    values: |
      key: value
destination:
  server: https://kubernetes.default.svc
  namespace: my-app
syncPolicy:
  syncOptions:
    - CreateNamespace=true
```

#### Path-Based (Manifests/Kustomize)

For applications with manifests in Git:

```yaml
name: my-app
project: default
source:
  repoURL: git@github.com:seanmccorkle/microk8s-components.git
  targetRevision: main
  path: apps/my-app
  kustomize:
    version: v5.0.0
destination:
  server: https://kubernetes.default.svc
  namespace: my-app
```

### How It Works

1. ApplicationSet scans `apps/*/config.yaml` files every 3 minutes
2. Detects source type (chart vs path)
3. Generates ArgoCD Application
4. Deploys in sync-wave order

### Applications

| Name | Type | Source | Sync Wave | Status |
|------|------|--------|-----------|--------|
| external-secrets-operator | Helm | https://charts.external-secrets.io | 5 | Active |
| network-test | Path | apps/network-test | 10 | Active |
| shinobi | Path | apps/shinobi | - | Disabled |

**Documentation:**
- External Secrets Operator: `apps/external-secrets-operator/ESO.md`

### Adding a New Application

1. Create `apps/my-app/config.yaml`
2. Commit and push
3. Wait 3 minutes or force refresh:

```bash
kubectl annotate applicationset microk8s-components -n argocd \
  argocd.argoproj.io/refresh=normal --overwrite
```

### Verification

```bash
# List applications
kubectl get applications -n argocd

# Check application status
argocd app get external-secrets-operator

# View ApplicationSet
kubectl get applicationset microk8s-components -n argocd
```

