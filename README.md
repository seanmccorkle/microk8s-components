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

