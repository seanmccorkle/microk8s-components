# HashiCorp Vault with External Secrets Operator

This directory contains the configuration for HashiCorp Vault integrated with External Secrets Operator.

## Architecture

```
Vault Server (dev mode)
  │
  ├─ KV v2 Secrets Engine (mount: secret)
  ├─ Kubernetes Auth Method
  └─ Policy: external-secrets
      │
      └─ Role: external-secrets
          │
          ├─ Bound to ServiceAccount: external-secrets-vault
          └─ Namespace: external-secrets-system
              │
              └─ External Secrets Operator
                  │
                  ├─ ClusterSecretStore: vault-backend
                  └─ SecretStore: vault-backend (namespace-scoped)
                      │
                      └─ ExternalSecrets
                          └─ Creates/syncs Kubernetes Secrets
```

## Deployment

### Prerequisites

**Before deploying Vault, ensure:**

1. **External Secrets Operator is deployed and running**
   ```bash
   kubectl get pods -n external-secrets-system
   # Should show: external-secrets-* pods Running
   ```

2. **ESO CRDs are installed**
   ```bash
   kubectl get crds | grep -E "(clustersecretstore|secretstore)"
   ```
   
   If missing, install them:
   ```bash
   kubectl apply --server-side --force-conflicts \
     -f https://raw.githubusercontent.com/external-secrets/external-secrets/main/deploy/crds/bundle.yaml
   ```

### Deployment Behavior

Resources deploy in **sync-wave order** (controlled by `argocd.argoproj.io/sync-wave` annotations):

```
ApplicationSet discovers vault/config.yaml
  └─> Creates Application "vault"
      │
      ├─> Wave 0 (Default): Vault Helm chart
      │   └─> Deployment, Service, PVC, ConfigMaps, etc.
      │   └─> Waits for Vault pod to be Ready
      │
      ├─> Wave 1: RBAC and ServiceAccount
      │   ├─> ServiceAccount (external-secrets-vault)
      │   ├─> ClusterRole (vault-token-reviewer)
      │   └─> ClusterRoleBinding
      │
      ├─> Wave 2: Init Scripts
      │   └─> ConfigMap (vault-init-scripts)
      │
      ├─> Wave 3: Vault Configuration
      │   └─> Job (vault-init) - Configures Vault
      │   └─> Waits for Job to Complete successfully
      │
      └─> Wave 4: ESO Integration
          └─> ClusterSecretStore (vault-backend)
          └─> Connects to configured Vault
```

**Sync-Wave Behavior:**
- ArgoCD processes waves sequentially (0 → 1 → 2 → 3 → 4)
- Waits for all resources in wave N to be healthy before starting wave N+1
- Resources within the same wave deploy in parallel
- Jobs must complete successfully before proceeding to next wave

**Expected Behavior:**
- Resources deploy in order without race conditions
- Vault pod starts and becomes ready (wave 0)
- RBAC and ServiceAccount created (wave 1)
- Init scripts ConfigMap ready (wave 2)
- Init job runs and completes (wave 3) - ArgoCD waits for completion
- ClusterSecretStore deploys after Vault is configured (wave 4)
- **ClusterSecretStore may still show errors if:**
  - ESO CRDs don't exist (application-level dependency)
  - ESO operator isn't running
  
  ArgoCD will retry and succeed once dependencies are met.

### Manual Deployment Steps

1. **Verify prerequisites:**
   ```bash
   # ESO running
   kubectl get deployment -n external-secrets-system external-secrets
   
   # CRDs exist
   kubectl get crds | grep -E "(clustersecretstore|secretstore)"
   ```

2. **Vault Application is auto-discovered:**
   ```bash
   # Check ApplicationSet found it
   kubectl get application vault -n argocd
   ```

3. **Sync the application:**
   ```bash
   argocd app sync vault
   ```

4. **Monitor deployment:**
   ```bash
   # Watch Vault pod
   kubectl get pods -n vault -w
   
   # Check init job
   kubectl get jobs -n vault
   kubectl logs -n vault job/vault-init
   
   # Check ClusterSecretStore
   kubectl get clustersecretstore vault-backend
   ```

## Accessing Vault

### Port Forward

```bash
kubectl port-forward -n vault svc/vault 8200:8200
```

### Using Vault CLI

```bash
export VAULT_ADDR='http://localhost:8200'
export VAULT_TOKEN='root'  # Dev mode token

# List secrets
vault kv list secret/

# Get a secret
vault kv get secret/database/config

# Put a secret
vault kv put secret/myapp/config \
  key1=value1 \
  key2=value2

# Delete a secret
vault kv delete secret/myapp/config
```

### Using kubectl exec

```bash
kubectl exec -n vault vault-0 -- vault status

# Set token and run commands
kubectl exec -n vault vault-0 -- sh -c '
  export VAULT_TOKEN=root
  vault kv list secret/
'
```

## Managing Secrets

### Example: Database Credentials

```bash
# Create secret in Vault
vault kv put secret/database/config \
  username="dbuser" \
  password="supersecret" \
  host="postgres.default.svc.cluster.local" \
  port="5432" \
  database="myapp"
```

Create an ExternalSecret to sync:

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: database-credentials
  namespace: default
spec:
  refreshInterval: 15m
  secretStoreRef:
    name: vault-backend
    kind: ClusterSecretStore
  target:
    name: database-secret
    creationPolicy: Owner
  data:
    - secretKey: username
      remoteRef:
        key: database/config
        property: username
    - secretKey: password
      remoteRef:
        key: database/config
        property: password
    - secretKey: host
      remoteRef:
        key: database/config
        property: host
    - secretKey: port
      remoteRef:
        key: database/config
        property: port
```

Verify the Kubernetes secret:

```bash
kubectl get secret database-secret -n default
kubectl get secret database-secret -n default -o yaml
```

### Example: API Keys (Sync Entire Secret)

```bash
# Create API keys in Vault
vault kv put secret/api/keys \
  openai="sk-test-key-123" \
  stripe="sk_test_key_456" \
  github="ghp_test_key_789"
```

Create an ExternalSecret:

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: api-keys
  namespace: default
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault-backend
    kind: ClusterSecretStore
  target:
    name: api-keys-secret
    creationPolicy: Owner
  dataFrom:
    - extract:
        key: api/keys  # Syncs all key-value pairs
```

This creates a Kubernetes secret with all keys from `secret/api/keys`:

```bash
kubectl get secret api-keys-secret -n default -o jsonpath='{.data.openai}' | base64 -d
```

### Example: Templated Secrets

Create a connection string from individual secret components:

```bash
# Create secrets in Vault
vault kv put secret/database/config \
  username="dbuser" \
  password="secret123" \
  host="postgres.default.svc.cluster.local" \
  port="5432" \
  database="myapp"

vault kv put secret/app/config \
  api_url="https://api.example.com" \
  log_level="info"
```

Create an ExternalSecret with templating:

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: app-config
  namespace: default
spec:
  refreshInterval: 30m
  secretStoreRef:
    name: vault-backend
    kind: ClusterSecretStore
  target:
    name: app-config-secret
    creationPolicy: Owner
    template:
      engineVersion: v2
      data:
        # Create a connection string from components
        database_url: "postgresql://{{ .db_username }}:{{ .db_password }}@{{ .db_host }}:{{ .db_port }}/{{ .db_name }}"
        api_url: "{{ .api_url }}"
        log_level: "{{ .log_level }}"
  data:
    - secretKey: db_username
      remoteRef:
        key: database/config
        property: username
    - secretKey: db_password
      remoteRef:
        key: database/config
        property: password
    - secretKey: db_host
      remoteRef:
        key: database/config
        property: host
    - secretKey: db_port
      remoteRef:
        key: database/config
        property: port
    - secretKey: db_name
      remoteRef:
        key: database/config
        property: database
    - secretKey: api_url
      remoteRef:
        key: app/config
        property: api_url
    - secretKey: log_level
      remoteRef:
        key: app/config
        property: log_level
```

The resulting secret will have:

```bash
kubectl get secret app-config-secret -n default -o jsonpath='{.data.database_url}' | base64 -d
# Output: postgresql://dbuser:secret123@postgres.default.svc.cluster.local:5432/myapp
```

## Using Secrets in Applications

### Environment Variables

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
  namespace: default
spec:
  containers:
  - name: app
    image: myapp:latest
    env:
      - name: DATABASE_USERNAME
        valueFrom:
          secretKeyRef:
            name: database-secret
            key: username
      - name: DATABASE_PASSWORD
        valueFrom:
          secretKeyRef:
            name: database-secret
            key: password
      - name: DATABASE_URL
        valueFrom:
          secretKeyRef:
            name: app-config-secret
            key: database_url
```

### Volume Mounts

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
  namespace: default
spec:
  containers:
  - name: app
    image: myapp:latest
    volumeMounts:
      - name: secrets
        mountPath: /etc/secrets
        readOnly: true
  volumes:
    - name: secrets
      secret:
        secretName: api-keys-secret
```

Access files:
- `/etc/secrets/openai`
- `/etc/secrets/stripe`
- `/etc/secrets/github`

## Troubleshooting

### Check ExternalSecret Status

```bash
kubectl get externalsecret -n default
kubectl describe externalsecret database-credentials -n default
```

### Check ESO Logs

```bash
kubectl logs -n external-secrets-system -l app.kubernetes.io/name=external-secrets --tail=50
```

### Check Vault Logs

```bash
kubectl logs -n vault vault-0 --tail=50
```

### Check Vault Status

```bash
kubectl exec -n vault vault-0 -- vault status
```

### Test Vault Connectivity from ESO

```bash
# Get ESO pod name
ESO_POD=$(kubectl get pod -n external-secrets-system -l app.kubernetes.io/name=external-secrets -o jsonpath='{.items[0].metadata.name}')

# Test connection
kubectl exec -n external-secrets-system $ESO_POD -- wget -O- http://vault.vault.svc.cluster.local:8200/v1/sys/health
```

### Verify Kubernetes Auth Configuration

```bash
kubectl exec -n vault vault-0 -- sh -c '
  export VAULT_TOKEN=root
  vault read auth/kubernetes/config
  vault read auth/kubernetes/role/external-secrets
'
```

### Force Secret Refresh

```bash
# Annotate to trigger immediate sync
kubectl annotate externalsecret database-credentials -n default \
  force-sync="$(date +%s)" --overwrite
```

## Important Notes

### Dev Mode Only

**⚠️ WARNING:** This configuration uses Vault in **development mode**:
- **In-memory storage** - Data is lost on pod restart
- **Root token:** `root` - Hardcoded and insecure
- **TLS disabled** - Unencrypted communication
- **Auto-unsealed** - No unseal process required
- **Single replica** - No high availability

**DO NOT USE THIS CONFIGURATION IN PRODUCTION**

For production, see commented configuration in `values.yaml`:
- Enable HA mode with Raft storage
- Configure persistent volumes
- Enable TLS with proper certificates
- Use proper initialization and unsealing procedures
- Configure auto-unseal with cloud KMS
- Remove hardcoded root token
- Implement proper secret rotation

### Cross-Namespace Resource

The `external-secrets-vault` ServiceAccount is created in the `external-secrets-system` namespace
but is part of the Vault Application (deployed to `vault` namespace). This is intentional for ESO
integration but creates a cross-namespace dependency. If you delete the Vault Application, this
ServiceAccount in external-secrets-system will also be deleted.

## Production Considerations

### 1. Remove Dev Mode

For production deployment, update `values.yaml`:

### 2. Secret Rotation

Configure refresh intervals based on sensitivity:
- Highly sensitive: 5-15 minutes
- Moderate: 1 hour
- Low sensitivity: 24 hours

### 3. RBAC and Policies

Create fine-grained policies per application:

```bash
# App-specific policy
vault policy write myapp-policy - <<EOF
path "secret/data/myapp/*" {
  capabilities = ["read", "list"]
}
EOF

# App-specific role
vault write auth/kubernetes/role/myapp \
  bound_service_account_names=myapp-sa \
  bound_service_account_namespaces=myapp-namespace \
  policies=myapp-policy \
  ttl=1h
```

### 4. Monitoring

Monitor:
- ExternalSecret sync status
- Secret refresh failures
- Vault health and performance
- Token expiry

### 5. Backup

Backup Vault data regularly:

```bash
# Create snapshot (Raft storage)
kubectl exec -n vault vault-0 -- vault operator raft snapshot save /tmp/backup.snap

# Copy from pod
kubectl cp vault/vault-0:/tmp/backup.snap ./vault-backup-$(date +%Y%m%d).snap
```

## References

- [Vault Documentation](https://www.vaultproject.io/docs)
- [Vault Kubernetes Auth](https://www.vaultproject.io/docs/auth/kubernetes)
- [External Secrets Vault Provider](https://external-secrets.io/latest/provider/hashicorp-vault/)
- [Vault Helm Chart](https://github.com/hashicorp/vault-helm)
