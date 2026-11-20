# External Secrets Operator Installation

This document provides installation instructions for External Secrets Operator.

## Installation Methods

### Using Official Manifests

The External Secrets Operator provides official installation manifests that are maintained by the project.

#### Install CRDs

```bash
kubectl apply -f https://raw.githubusercontent.com/external-secrets/external-secrets/main/deploy/crds/bundle.yaml
```

Wait for CRDs to be established:

```bash
kubectl wait --for=condition=established --timeout=60s crd/externalsecrets.external-secrets.io
```

#### Install Operator

```bash
kubectl apply -f https://raw.githubusercontent.com/external-secrets/external-secrets/main/bundle.yaml
```

### Using Specific Version

To install a specific version instead of the latest:

```bash
VERSION=v0.9.11

# Install CRDs
kubectl apply -f https://raw.githubusercontent.com/external-secrets/external-secrets/${VERSION}/deploy/crds/bundle.yaml

# Install operator
kubectl apply -f https://raw.githubusercontent.com/external-secrets/external-secrets/${VERSION}/bundle.yaml
```

### Using Helm (Recommended)

```bash
helm repo add external-secrets https://charts.external-secrets.io
helm repo update

helm install external-secrets \
  external-secrets/external-secrets \
  -n external-secrets-system \
  --create-namespace
```

## Verification

Check that the operator is running:

```bash
kubectl get pods -n external-secrets-system
```

Verify CRDs are installed:

```bash
kubectl get crds | grep external-secrets.io
```

Expected CRDs:
- `clustersecretstores.external-secrets.io`
- `externalsecrets.external-secrets.io`
- `secretstores.external-secrets.io`

## Usage Examples

### Basic Concepts

External Secrets Operator has two main resource types:

1. **SecretStore/ClusterSecretStore**: Defines HOW to connect to an external secret provider
   - Namespace-scoped (`SecretStore`) or cluster-scoped (`ClusterSecretStore`)
   - Contains authentication configuration
   - Multiple stores can exist for different providers or environments

2. **ExternalSecret**: Defines WHAT secrets to sync and WHERE they come from
   - References a SecretStore
   - Specifies which secrets to pull from the external provider
   - Creates/updates native Kubernetes Secrets

### Testing with Fake Provider

Use the fake provider for development and testing:

```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: fake-store
  namespace: default
spec:
  provider:
    fake:
      data:
        - key: "test/api-key"
          value: "fake-api-key-123"
          version: "v1"
        - key: "test/database"
          value: '{"username": "admin", "password": "changeme"}'
          version: "v1"
```

Create an ExternalSecret to sync from the fake provider:

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: example-fake-secret
  namespace: default
spec:
  refreshInterval: 5m
  secretStoreRef:
    name: fake-store
    kind: SecretStore
  target:
    name: my-synced-secret
    creationPolicy: Owner
    deletionPolicy: Retain
  data:
    - secretKey: api_key
      remoteRef:
        key: test/api-key
    - secretKey: db_username
      remoteRef:
        key: test/database
        property: username
    - secretKey: db_password
      remoteRef:
        key: test/database
        property: password
```

### Templating

Transform secret data using Go templates:

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: example-templated-secret
  namespace: default
spec:
  refreshInterval: 5m
  secretStoreRef:
    name: fake-store
    kind: SecretStore
  target:
    name: templated-secret
    template:
      engineVersion: v2
      data:
        database_url: "postgresql://{{ .username }}:{{ .password }}@db.example.com:5432/mydb"
  data:
    - secretKey: username
      remoteRef:
        key: test/database
        property: username
    - secretKey: password
      remoteRef:
        key: test/database
        property: password
```

### ClusterSecretStore

Create a cluster-wide SecretStore accessible from any namespace:

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: cluster-fake-store
spec:
  provider:
    fake:
      data:
        - key: "shared/api-key"
          value: "shared-api-key-456"
          version: "v1"
```

Use it from any namespace:

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: example-cluster-secret
  namespace: default
spec:
  refreshInterval: 5m
  secretStoreRef:
    name: cluster-fake-store
    kind: ClusterSecretStore
  target:
    name: cluster-synced-secret
  data:
    - secretKey: shared_api_key
      remoteRef:
        key: shared/api-key
```

## Provider Examples

### AWS Secrets Manager

**Prerequisites:**
- AWS credentials with Secrets Manager access
- Or IAM Role for Service Accounts (IRSA) on EKS

Create AWS credentials secret:
```bash
kubectl create secret generic aws-credentials \
  --from-literal=access-key-id=YOUR_ACCESS_KEY \
  --from-literal=secret-access-key=YOUR_SECRET_KEY \
  -n your-namespace
```

**SecretStore with static credentials:**

```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: aws-secretsmanager
  namespace: default
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-east-1
      auth:
        secretRef:
          accessKeyIDSecretRef:
            name: aws-credentials
            key: access-key-id
          secretAccessKeySecretRef:
            name: aws-credentials
            key: secret-access-key
```

**SecretStore with IRSA (recommended for EKS):**

```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: aws-secretsmanager-irsa
  namespace: default
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-east-1
      auth:
        jwt:
          serviceAccountRef:
            name: external-secrets-sa
```

**ExternalSecret example:**

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: example-aws-secret
  namespace: default
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secretsmanager
    kind: SecretStore
  target:
    name: my-app-secret
    creationPolicy: Owner
  data:
    - secretKey: username
      remoteRef:
        key: my-app/database
        property: username
    - secretKey: password
      remoteRef:
        key: my-app/database
        property: password
```

**Sync entire JSON secret:**

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: example-aws-json-secret
  namespace: default
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secretsmanager
    kind: SecretStore
  target:
    name: my-app-config
    creationPolicy: Owner
  dataFrom:
    - extract:
        key: my-app/config
```

### Google Secret Manager

**Prerequisites:**
- GCP Service Account with Secret Manager access
- Or Workload Identity on GKE

Create GCP credentials secret:
```bash
kubectl create secret generic gcp-credentials \
  --from-file=credentials.json=./gcp-key.json \
  -n your-namespace
```

**SecretStore with service account key:**

```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: gcp-secret-manager
  namespace: default
spec:
  provider:
    gcpsm:
      projectID: "your-gcp-project-id"
      auth:
        secretRef:
          secretAccessKeySecretRef:
            name: gcp-credentials
            key: credentials.json
```

**SecretStore with Workload Identity (recommended for GKE):**

```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: gcp-secret-manager-wi
  namespace: default
spec:
  provider:
    gcpsm:
      projectID: "your-gcp-project-id"
      auth:
        workloadIdentity:
          clusterLocation: us-central1
          clusterName: my-cluster
          serviceAccountRef:
            name: external-secrets-sa
```

**ExternalSecret example:**

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: example-gcp-secret
  namespace: default
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: gcp-secret-manager
    kind: SecretStore
  target:
    name: gcp-synced-secret
    creationPolicy: Owner
  data:
    - secretKey: api-key
      remoteRef:
        key: my-app-api-key
        version: latest
    - secretKey: database-url
      remoteRef:
        key: my-app-database-url
        version: latest
```

**Specific version:**

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: example-gcp-versioned-secret
  namespace: default
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: gcp-secret-manager
    kind: SecretStore
  target:
    name: gcp-versioned-secret
    creationPolicy: Owner
  data:
    - secretKey: config
      remoteRef:
        key: my-app-config
        version: "3"
```

## References

- [Official Installation Guide](https://external-secrets.io/latest/introduction/getting-started/)
- [GitHub Releases](https://github.com/external-secrets/external-secrets/releases)
- [Helm Chart Repository](https://github.com/external-secrets/external-secrets/tree/main/deploy/charts/external-secrets)
- [Provider Documentation](https://external-secrets.io/latest/provider/aws-secrets-manager/)
