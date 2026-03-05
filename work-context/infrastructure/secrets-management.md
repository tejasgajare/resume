# Secrets Management — SOPS, ESO, Vault Migration

> **Tags**: `#interview-ready` `#secrets` `#sops` `#eso` `#external-secrets` `#vault` `#aws-parameter-store` `#kms`
> **Role**: Software Engineer — RAVE Platform, HPE GreenLake
> **Jira**: GLCP-325064 (custom service account on rave-prod)
> **Cross-references**: [aws-terraform.md](./aws-terraform.md), [kubernetes-helm.md](./kubernetes-helm.md), [cicd-pipelines.md](./cicd-pipelines.md)

---

## Overview

I own the secrets management infrastructure for the RAVE platform, spanning three complementary systems: SOPS for Git-encrypted secrets, External Secrets Operator (ESO) for Kubernetes-native secret synchronization from AWS Parameter Store, and the ongoing migration away from HashiCorp Vault. This is one of my most significant infrastructure contributions — I designed the ESO architecture, implemented the migration for the first three services, and am driving the remaining 17 services through the transition.

---

## Secrets Architecture — Before & After

### Before (Vault-based)

Previously, RAVE secrets were stored in HashiCorp Vault, deployed within the HPE GreenLake platform:
- Vault stored MongoDB credentials, SendGrid API keys, SDI client secrets
- Services accessed Vault via sidecar injectors or init containers
- Vault required its own infrastructure: HA cluster, unsealing, audit logging
- The CCP team mandated moving off Vault due to licensing and operational overhead

### After (SOPS + ESO + AWS Parameter Store)

The new architecture:
```
Developer encrypts secret with SOPS (using AWS KMS)
  → Encrypted YAML committed to rave-sops-{env} repo
    → CI/CD pipeline decrypts and pushes to AWS Parameter Store
      → ESO SecretStore authenticates via IRSA ServiceAccount
        → ESO ExternalSecret CRD syncs Parameter Store → K8s Secret
          → Pod mounts K8s Secret as env var or volume
```

This eliminates Vault entirely and uses native AWS services (KMS, Parameter Store) with Kubernetes-native CRDs (ESO).

---

## SOPS Encryption

### What is SOPS?

SOPS (Secrets OPerationS) by Mozilla encrypts structured data files (YAML, JSON) while leaving the keys/structure readable. Only the values are encrypted, making diffs meaningful in Git.

### RAVE SOPS Repos

I manage three environment-specific SOPS repos:

| Repo | Environment | KMS Key Alias |
|------|-------------|---------------|
| `rave-sops-dev` | dev | `eso/rave/rave-dev/rave-dev-secrets` |
| `rave-sops-stage` | stage | `eso/rave/rave-stag/rave-stag-secrets` |
| `rave-sops-prod` | prod | `eso/rave/rave-prod/rave-prod-secrets` |

Each repo contains:
```
sops/
├── mongodb-credentials.yaml    # MongoDB Atlas connection strings + auth
├── sdiclientsecret.yaml        # SDI client OAuth secret
├── sendgrid-api-key.yaml       # SendGrid email API key
└── .gitkeep
```

### SOPS File Format

An encrypted SOPS file looks like this (`mongodb-credentials.yaml`):
```yaml
name: rave/mongodb/credentials
data: ENC[AES256_GCM,data:W4KPpjszDA4jopZ...,tag:hfGJ5I//rU6ZNgU3LJvRGQ==,type:str]
rotation_runbook: https://hpe.atlassian.net/wiki/spaces/IN2/pages/2561769923/CCP+RAVE+Cluster
tags:
    Environment: dev
    Team: rave
sops:
    kms:
        - arn: arn:aws:kms:us-west-2:590183756478:alias/eso/rave/rave-dev/rave-dev-secrets
          created_at: "2026-01-14T03:17:10Z"
          enc: AQICAHhP19D/cnwr8GLcOm...
    lastmodified: "2026-01-14T03:18:06Z"
    mac: ENC[AES256_GCM,data:7xHVJ3kn22mhiRQ...,type:str]
    encrypted_regex: ^(data)$
    version: 3.11.0
```

Key design decisions:
- **`encrypted_regex: ^(data)$`**: Only the `data` field is encrypted. Metadata (`name`, `tags`, `rotation_runbook`) stays readable, making PRs reviewable and secrets auditable
- **KMS ARN per environment**: Each repo uses a different KMS key, so dev credentials can't decrypt prod secrets even if someone copies the file
- **Rotation runbook link**: Each secret includes a link to the rotation procedure in Confluence
- **MAC verification**: The `mac` field ensures the entire file hasn't been tampered with

### My SOPS Workflow

When I need to rotate or add a secret:

```bash
# 1. Configure AWS credentials for the target environment
aws sso login --profile rave-dev

# 2. Decrypt the file for editing
sops -d sops/mongodb-credentials.yaml > /tmp/mongodb-plain.yaml

# 3. Edit the plaintext file
vim /tmp/mongodb-plain.yaml

# 4. Re-encrypt with SOPS
sops -e /tmp/mongodb-plain.yaml > sops/mongodb-credentials.yaml

# 5. Verify the encryption
sops -d sops/mongodb-credentials.yaml  # should show plaintext

# 6. Commit and push
git add sops/mongodb-credentials.yaml
git commit -m "Rotate MongoDB credentials"
git push

# 7. Clean up
rm /tmp/mongodb-plain.yaml
```

The `.sops.yaml` creation rules in each repo automatically select the correct KMS key, so I don't need to specify it manually.

---

## External Secrets Operator (ESO)

### Architecture

ESO bridges AWS Parameter Store and Kubernetes Secrets. I designed and implemented this for RAVE:

```
AWS Parameter Store                    Kubernetes
─────────────────                    ──────────
/eso/rave/rave-dev/rave-dev-secrets/
  ├── rave/mongodb/credentials    ──→  Secret: rave-mongodb-secret
  ├── rave/sendgrid/api-key       ──→  Secret: rave-sendgrid-secret
  └── rave/sdi/clientsecret       ──→  Secret: rave-sdiclient-secret

        ↑ authenticated via                ↑ created by
  ClusterSecretStore (IRSA)         ExternalSecret CRDs
```

### ClusterSecretStore

The ClusterSecretStore is a cluster-wide resource (managed by CCP) that defines how ESO authenticates with AWS:

```yaml
apiVersion: external-secrets.io/v1
kind: ClusterSecretStore
metadata:
  name: rave-dev-secrets
spec:
  provider:
    aws:
      service: ParameterStore
      region: us-west-2
      auth:
        jwt:
          serviceAccountRef:
            name: eso-service-account
            namespace: rave
```

The ServiceAccount `eso-service-account` has an IRSA annotation binding it to an IAM role with `ssm:GetParameter` permissions on the `/eso/rave/` path prefix.

### ExternalSecret CRDs

I implemented the ExternalSecret templates in the Helm chart (`shared-externalsecrets.yaml`):

```yaml
{{- if .Values.eso.enabled }}
---
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: rave-sendgrid
  namespace: {{ .Values.global.namespace }}
  labels:
    {{ include "helm-rave.labels" . | nindent 4 }}
    app: shared-secrets
spec:
  secretStoreRef:
    name: {{ .Values.eso.clusterSecretStore }}
    kind: ClusterSecretStore

  target:
    name: {{ .Values.eso.secretNames.sendgrid }}
    creationPolicy: Owner

  refreshInterval: {{ .Values.eso.refreshInterval | default "1h" }}

  data:
    - secretKey: sendgrid.json
      remoteRef:
        key: {{ .Values.eso.parameterStorePath }}/rave/sendgrid/api-key
---
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: rave-mongodb
  namespace: {{ .Values.global.namespace }}
spec:
  secretStoreRef:
    name: {{ .Values.eso.clusterSecretStore }}
    kind: ClusterSecretStore

  target:
    name: {{ .Values.eso.secretNames.mongodb }}
    creationPolicy: Owner

  refreshInterval: {{ .Values.eso.refreshInterval | default "1h" }}

  data:
    - secretKey: mongodb.json
      remoteRef:
        key: {{ .Values.eso.parameterStorePath }}/rave/mongodb/credentials
---
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: rave-sdiclient
  namespace: {{ .Values.global.namespace }}
spec:
  secretStoreRef:
    name: {{ .Values.eso.clusterSecretStore }}
    kind: ClusterSecretStore

  target:
    name: {{ .Values.eso.secretNames.sdiclient }}
    creationPolicy: Owner

  refreshInterval: {{ .Values.eso.refreshInterval | default "1h" }}

  data:
    - secretKey: sdiclientsecret.json
      remoteRef:
        key: {{ .Values.eso.parameterStorePath }}/rave/sdi/clientsecret
{{- end }}
```

### ESO Configuration in Helm Values

Each environment configures ESO via the values file:

```yaml
# values-dev.yaml
eso:
  enabled: true
  clusterSecretStore: rave-dev-secrets
  parameterStorePath: /eso/rave/rave-dev/rave-dev-secrets
  refreshInterval: 1h
  secretNames:
    mongodb: rave-mongodb-secret
    sendgrid: rave-sendgrid-secret
    sdiclient: rave-sdiclient-secret

# values-prod.yaml
eso:
  enabled: true
  clusterSecretStore: rave-prod-secrets
  parameterStorePath: /eso/rave/rave-prod/rave-prod-secrets
  refreshInterval: 1h
  secretNames:
    mongodb: rave-mongodb-secret
    sendgrid: rave-sendgrid-secret
    sdiclient: rave-sdiclient-secret
```

The `refreshInterval: 1h` means ESO checks Parameter Store hourly for changes. If a secret is rotated in Parameter Store, the Kubernetes Secret updates automatically within an hour—no pod restart needed (if the app watches for file changes) or within the next pod restart cycle.

---

## ESO Migration from Vault

### Migration Status

**Completed (3 services)**:
| Service | Secret Type | Status | Notes |
|---------|------------|--------|-------|
| caseRelay | MongoDB, SendGrid | ✅ Done | First migration, validated pattern |
| crmRelay | MongoDB, SDI client | ✅ Done | High-traffic service, validated under load |
| wellnessProducer | MongoDB | ✅ Done | Kafka producer, validated async secret access |

**Remaining (17 services)**:
- backtracefetchservice, correlationengine, crmservice, dataingestion, dataparser, dbupdate, decisionengine, deviceshadow, eventsDataProvider, glpnotificationconsumer, healthchecker, homefleetcollector, pointnextcrm, comenricher, alertCollector, alertHandler, arcusCollector

### Migration Process Per Service

For each service, the migration involves:

1. **Identify secrets**: What secrets does this service consume? (MongoDB, API keys, OAuth tokens)
2. **Add to Parameter Store**: Ensure the secret exists at the correct path in AWS Parameter Store
3. **Add ExternalSecret CRD**: If it's a new secret type, add to `shared-externalsecrets.yaml`
4. **Update deployment template**: Change the secret volume mount or env var reference from the Vault secret name to the ESO secret name
5. **Test in dev**: Deploy to dev, verify the service reads the correct secret
6. **Promote to stage/prod**: Update values files for each environment
7. **Remove Vault references**: Clean up Vault annotations, init containers, sidecar configs

### Custom ServiceAccount Work (GLCP-325064)

The prod environment requires a custom ServiceAccount for ESO authentication because the default SA lacks AWS IAM bindings. My Jira ticket GLCP-325064 covers:

**Problem**: In dev, we could use a shared ServiceAccount with broad Parameter Store access. In prod, security requires per-service (or per-namespace) scoped IAM roles.

**Solution**: I created a dedicated ServiceAccount for the `rave` namespace in prod:
1. ServiceAccount with IRSA annotation pointing to an IAM role with `ssm:GetParameter` on `/eso/rave/rave-prod/*`
2. The ClusterSecretStore references this ServiceAccount
3. The IAM role has a trust policy scoped to the `rave` namespace's OIDC provider

**Why this matters**: Without this, prod pods would need to use the node instance role (too broad) or inline AWS credentials (insecure). The custom SA provides least-privilege access — only the ESO controller in the `rave` namespace can read RAVE's prod secrets.

---

## Secret Types & Mapping

### Current Secrets

| Secret Name | Parameter Store Path | Used By | Content |
|-------------|---------------------|---------|---------|
| `rave-mongodb-secret` | `/eso/rave/{env}/rave-{env}-secrets/rave/mongodb/credentials` | All DB-connected services | MongoDB connection URI + credentials |
| `rave-sendgrid-secret` | `/eso/rave/{env}/rave-{env}-secrets/rave/sendgrid/api-key` | caserelay, notification services | SendGrid API key for email |
| `rave-sdiclient-secret` | `/eso/rave/{env}/rave-{env}-secrets/rave/sdi/clientsecret` | crmrelay, crmservice | SDI OAuth client secret |
| `pagerduty-secret` | *(hardcoded in common-secrets.yaml)* | AlertManager | PagerDuty routing key |

### Secret Consumption Pattern

Services consume secrets via environment variables or volume mounts:

```yaml
env:
  - name: MONGODB_URI
    valueFrom:
      secretKeyRef:
        name: rave-mongodb-secret
        key: mongodb.json
```

The `mongodb.json` key matches the `secretKey` in the ExternalSecret's `data` spec, which maps to the Parameter Store parameter key.

---

## Security Considerations

### Encryption at Rest
- **SOPS**: AES-256-GCM encryption, KMS-managed keys per environment
- **Parameter Store**: AWS-managed encryption (also KMS-backed)
- **Kubernetes Secrets**: etcd encryption at rest (managed by EKS)

### Encryption in Transit
- **ESO → Parameter Store**: TLS via AWS SDK
- **Pod → Secret**: In-memory mount (no network transit)
- **SOPS → Parameter Store**: TLS via CI pipeline

### Access Control
- **KMS keys**: Separate per environment, IAM policy controls who can encrypt/decrypt
- **Parameter Store**: IAM path-based access (`/eso/rave/rave-{env}/*`)
- **Kubernetes RBAC**: Only the ESO controller can create/update secrets in the `rave` namespace
- **IRSA**: Pod-level AWS access scoped to specific IAM roles

### Rotation Strategy
- **MongoDB**: Rotated via SOPS → Parameter Store → ESO sync (manual trigger, automated propagation)
- **SendGrid**: Annual rotation, documented in Confluence runbook
- **SDI client**: Rotated when OAuth provider requires
- **PagerDuty**: Static routing key (low-risk, monitored)

---

## Interview Deep-Dives

### Q: Why migrate from Vault to ESO + Parameter Store?

HashiCorp changed their licensing model (BSL), and HPE's platform team decided to move off Vault organization-wide. ESO + Parameter Store is a better fit for our Kubernetes-native architecture anyway: ESO is a CRD-based operator that syncs secrets from any external store into Kubernetes Secrets, and AWS Parameter Store is a managed service with no operational overhead (no HA cluster, no unsealing ceremony, no backup/restore). The migration reduces infrastructure complexity and eliminates a single point of failure.

### Q: Walk me through the ESO secret sync flow.

The ESO controller runs in the cluster and watches for ExternalSecret CRDs. When it finds one, it reads the `secretStoreRef` to know how to authenticate (in our case, via a ClusterSecretStore using IRSA). It then calls AWS Parameter Store's `GetParameter` API with the path from `remoteRef.key`. The returned value is stored as a Kubernetes Secret with the name and key specified in `target`. The controller re-syncs at `refreshInterval` (1 hour for us). If the Parameter Store value changes, the Kubernetes Secret is updated automatically.

### Q: How do you handle secret rotation without downtime?

ESO's `refreshInterval` means Kubernetes Secrets update automatically when Parameter Store changes. For services that read secrets at startup (most of ours), we trigger a rolling restart after rotation: `kubectl rollout restart deployment/<service>`. For services that could read secrets dynamically, we'd mount the secret as a volume file and watch for file changes. Currently, we do rolling restarts for simplicity and reliability.

### Q: What are the failure modes?

1. **ESO controller down**: Existing Kubernetes Secrets remain valid, but new changes won't sync. Pods continue running with stale secrets.
2. **Parameter Store unavailable**: Same as above — ESO can't refresh, but existing secrets work.
3. **IAM role misconfiguration**: ESO can't authenticate → ExternalSecret enters `SecretSyncedError` state. Detected via monitoring.
4. **KMS key deletion**: SOPS files become undecryptable. We have key rotation policies, not deletion policies.
5. **SOPS file corruption**: The MAC verification catches tampering. We keep secrets in separate files to minimize blast radius.

### Q: What would you do differently?

I'd implement a secret rotation automation pipeline: a GitHub Actions workflow that rotates MongoDB credentials on a schedule—generates new credentials, updates Parameter Store, triggers ESO sync, validates service health, then deactivates old credentials. Currently, rotation is manual. I'd also implement `ExternalSecret` status monitoring in our Prometheus/PagerDuty stack to alert on sync failures.

---

## Key Repos

| Repo | Purpose | My Role |
|------|---------|---------|
| `rave-sops-dev` | Dev secrets (SOPS-encrypted) | Encrypt/decrypt/rotate secrets |
| `rave-sops-stage` | Stage secrets | Same as dev |
| `rave-sops-prod` | Prod secrets | Same, with stricter review |
| `rave-cloud` | Helm chart with ESO templates | Designed shared-externalsecrets.yaml |
| `mcd-deploy-proj-rave` | ESO config in values files | Environment-specific ESO settings |
| `aws-wellness_*-config` | Parameter Store + IAM | Define Parameter Store paths, IAM roles |

---

## Metrics & Impact

- **3 services migrated** from Vault to ESO (caseRelay, crmRelay, wellnessProducer)
- **17 services remaining** in the migration pipeline
- **3 SOPS repos** with KMS-encrypted secrets across environments
- **Zero secret exposure incidents** — no credentials in code, no long-lived tokens in pods
- **1-hour refresh interval** for automatic secret synchronization
- **Eliminated Vault dependency** for migrated services (reduced operational overhead)
