# Multi-Environment Infrastructure & ESO Migration

## STAR Narrative

### Situation
The RAVE Cloud Platform and Nimble Download services at HPE GreenLake operated across three environments: dev, integration (intg), and production (prod), all running on Kubernetes. Secrets management was handled by HashiCorp Vault, but the setup had accumulated technical debt: Vault sidecar injectors added latency to pod startup, secret rotation required pod restarts, and the Vault configuration was tightly coupled to the deployment pipeline. With 20 microservices across the platform, managing Vault policies, roles, and paths for each service in each environment was becoming unsustainable. Additionally, the team was adopting AWS-native services, and secrets stored in AWS Parameter Store couldn't be easily consumed by pods using the Vault-only workflow. SOPS (Secrets OPerationS) encryption was used for secrets in Git, but the workflow was inconsistent across teams.

### Task
I was responsible for:
1. Managing the multi-environment infrastructure, ensuring dev/intg/prod parity while allowing environment-specific configurations (issuer domains, feature flags, resource limits).
2. Migrating secrets management from Vault to External Secrets Operator (ESO) — starting with 3 services as a pilot (caseRelay, crmRelay, wellnessProducer), with 17 more planned.
3. Implementing SOPS encryption for secrets stored in Git repositories.
4. Configuring AWS Parameter Store as the backing secret store via ESO.
5. Setting up custom ServiceAccounts with IAM role bindings for pod-level AWS access.

### Action
**Step 1 — ESO Architecture Design:**
I designed the ESO migration architecture with three Kubernetes CRDs:
- **SecretStore**: Cluster-scoped resource pointing to AWS Parameter Store in the correct region, authenticated via IAM roles for service accounts (IRSA).
- **ExternalSecret**: Per-service resource that maps AWS Parameter Store paths to Kubernetes Secret keys. For example, `/rave/{env}/wellnessProducer/db-password` maps to a k8s Secret with key `DB_PASSWORD`.
- **ClusterSecretStore** (optional): For shared secrets across namespaces (e.g., common TLS certificates).

The key design decision was using AWS Parameter Store over Secrets Manager because: (1) Parameter Store is free for standard parameters, (2) our secrets didn't need automatic rotation (Secrets Manager's main advantage), and (3) Parameter Store paths aligned naturally with our `/{service}/{env}/{key}` naming convention.

**Step 2 — IAM Role for Service Accounts (IRSA):**
Each service needed its own ServiceAccount annotated with an IAM role ARN:
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: wellnessproducer-sa
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::role/wellnessproducer-{env}
```
The IAM role policy was scoped to only the Parameter Store paths that service needed — least-privilege access. This was a significant improvement over the Vault approach where the sidecar injector had broader access.

**Step 3 — SOPS Encryption Workflow:**
For secrets that needed to live in Git (e.g., Helm values with sensitive defaults), I set up SOPS with AWS KMS:
- `.sops.yaml` configuration file at the repo root defining encryption rules per path pattern.
- Developers encrypt with `sops -e secrets.yaml > secrets.enc.yaml`.
- ArgoCD decrypts at deploy time using the `ksops` plugin (Kustomize + SOPS).
- Each environment has its own KMS key, so dev secrets can't be decrypted by prod keys and vice versa.

**Step 4 — Pilot Migration (3 Services):**
I migrated caseRelay, crmRelay, and wellnessProducer as the pilot:
1. Created ExternalSecret resources for each service.
2. Verified the generated Kubernetes Secrets matched the existing Vault-injected secrets.
3. Updated deployments to mount secrets from the ESO-generated Secrets instead of Vault annotations.
4. Removed Vault sidecar injector annotations.
5. Tested in dev → intg → prod with rollback plan (re-add Vault annotations if ESO fails).

**Step 5 — Multi-Environment Configuration Management:**
I structured Helm values with environment-specific overrides:
```
charts/
  wellnessProducer/
    values.yaml          # shared defaults
    values-dev.yaml      # dev overrides (issuer domains, replicas: 1)
    values-intg.yaml     # intg overrides (issuer domains, replicas: 2)
    values-prod.yaml     # prod overrides (issuer domains, replicas: 3, resource limits)
```
ArgoCD ApplicationSets selected the correct values file based on the target cluster/environment label, maintaining DRY configuration.

**Step 6 — Validation and Documentation:**
I wrote runbooks for: adding a new secret, rotating a secret, onboarding a new service to ESO, and emergency rollback to Vault. I also created a migration checklist template for the remaining 17 services.

### Result
- **3 services successfully migrated** from Vault to ESO (caseRelay, crmRelay, wellnessProducer) with zero downtime.
- **Pod startup time improved by ~2 seconds** — no Vault sidecar init container delay.
- **Least-privilege access** — each service can only read its own secrets, enforced by IAM policy.
- **Secret rotation simplified** — update the value in AWS Parameter Store, ESO syncs it to the k8s Secret within the configured `refreshInterval` (1 hour default), no pod restart needed.
- **SOPS workflow standardized** — consistent encryption across all repos, auditable in Git history.
- **Migration template** enabled other teams to self-service migrate their remaining 17 services.
- **Cost reduction** — eliminated Vault license costs for the RAVE workload.

## Interview Delivery Tips

### How to Open This Story
"I managed the multi-environment Kubernetes infrastructure for our platform and led the migration from HashiCorp Vault to External Secrets Operator. We completed 3 services as a pilot with 17 more planned. This involved IRSA for pod-level AWS access, SOPS encryption for secrets in Git, and ArgoCD-driven deployments."

### Time Budget (5-minute answer)
- Situation: 45 seconds (Vault pain points, AWS adoption, inconsistency)
- Task: 30 seconds (5 responsibilities)
- Action: 2.5 minutes (ESO architecture + IRSA are key technical depth)
- Result: 1 minute (zero downtime, faster startup, migration template)

### Pivot Points
- **If interviewer asks about Kubernetes**: Deep dive on CRDs, ServiceAccounts, IRSA
- **If interviewer asks about security**: Focus on least-privilege IAM, SOPS encryption, KMS key separation
- **If interviewer asks about DevOps/GitOps**: ArgoCD ApplicationSets, Helm values structure, drift detection
- **If interviewer asks about migration**: Pilot strategy, rollback plan, template for remaining services

---

## Rubric Annotations

### Key Phrases/Concepts That MUST Appear
- External Secrets Operator (ESO) with SecretStore/ExternalSecret CRDs
- AWS Parameter Store as the backing store
- IAM Roles for Service Accounts (IRSA)
- SOPS encryption with AWS KMS
- Pod-level least-privilege access
- Multi-environment Helm values structure
- ArgoCD deployment pipeline

### Common Mistakes
- Confusing ESO with Sealed Secrets (different approach entirely)
- Not explaining WHY Parameter Store over Secrets Manager
- Skipping the IRSA setup — this is the critical security piece
- Describing the migration without mentioning rollback strategy
- Not addressing secret rotation improvements

### Scoring Rubric [1-5]
- 1 — Poor: Cannot explain the difference between Vault and ESO or why migration was needed
- 2 — Below Average: Describes ESO at a high level but lacks IAM/IRSA details and migration strategy
- 3 — Acceptable: Covers ESO, IRSA, and SOPS but light on multi-environment management and trade-offs
- 4 — Strong: Full narrative including pilot strategy, rollback plan, and environment-specific config
- 5 — Exceptional: All of the above plus discusses cost implications, secret rotation lifecycle, and migration template for scale

---

## Follow-Up Questions (Progressively Harder)

### Follow-Up 1: How does ESO sync secrets from AWS Parameter Store to Kubernetes? What happens if the external store is unavailable?
**Expected Answer**: ESO runs a controller that periodically (per `refreshInterval`) reads from the external store and reconciles the Kubernetes Secret. If the external store is unavailable, the existing Kubernetes Secret remains unchanged — pods continue using the last-synced value. ESO sets a condition on the ExternalSecret resource (`SecretSyncedError`) so you can monitor sync failures. The Kubernetes Secret is the source of truth for running pods; ESO is the sync mechanism.
**Red Flags**: Says pods directly read from AWS Parameter Store at runtime.
**Green Flags**: Explains the reconciliation loop, mentions the Secret persists during outages, knows about status conditions.

### Follow-Up 2: What's the difference between IRSA and other methods of giving pods AWS access (e.g., node instance profile, kube2iam)?
**Expected Answer**: Node instance profile gives ALL pods on the node the same AWS permissions — no isolation. kube2iam intercepts metadata API calls and assumes roles per pod, but it's a third-party proxy with latency and reliability concerns. IRSA is the AWS-native solution: the ServiceAccount token is exchanged for AWS STS credentials via OIDC federation. Each pod gets its own IAM role scoped to its ServiceAccount, with no proxy and no shared credentials. IRSA is the recommended approach for EKS.
**Red Flags**: Doesn't know what IRSA stands for; confuses it with node roles.
**Green Flags**: Explains OIDC federation, contrasts with node profiles and kube2iam, mentions STS.

### Follow-Up 3: How do you handle secret rotation without restarting pods?
**Expected Answer**: ESO syncs the updated value to the Kubernetes Secret. However, pods that mount Secrets as environment variables won't see the change until restart. Pods that mount Secrets as files will see the updated file after kubelet's sync period (~1 minute). For zero-restart rotation: mount secrets as files and have the application watch for file changes, or use a sidecar that reloads config on file change. Alternatively, for database credentials, use IAM database authentication which uses short-lived tokens that are refreshed automatically.
**Red Flags**: Says "just restart the pods" as the only strategy.
**Green Flags**: Distinguishes between env var mounts (require restart) and file mounts (auto-update), mentions file watching.

---

## Grilling Chain (7 questions drilling deeper)

### Q1: Explain Kubernetes Secrets. How are they stored, and are they actually secure?
**Why asked**: Tests understanding of the K8s secrets model and its limitations.
**Expected answer**: Kubernetes Secrets are base64-encoded (NOT encrypted) by default and stored in etcd. Anyone with etcd access or `kubectl get secret` permissions can read them. To secure them: (1) Enable etcd encryption at rest (`EncryptionConfiguration`). (2) Use RBAC to restrict Secret access. (3) Enable audit logging for Secret reads. (4) Consider external secret stores (Vault, ESO) to minimize the window secrets exist in etcd. Base64 is encoding, not encryption — a common misconception.
**Red flags**: Says Kubernetes Secrets are encrypted by default; confuses base64 with encryption.
**Green flags**: Mentions etcd encryption at rest, RBAC, the base64 misconception.

### Q2: What is SOPS and how does it differ from Sealed Secrets?
**Why asked**: Tests knowledge of Git-based secret encryption approaches.
**Expected answer**: SOPS encrypts values within YAML/JSON files while leaving keys in plaintext — you can see the structure but not the values. It uses external KMS (AWS KMS, GCP KMS, Azure Key Vault, or PGP) for encryption. Sealed Secrets is a Kubernetes-native approach where you encrypt the entire Secret with the cluster's public key — only the target cluster can decrypt it. Key differences: SOPS is tool-agnostic (works outside K8s), supports partial encryption, and the KMS key can be rotated independently. Sealed Secrets is K8s-specific and the sealing key is per-cluster.
**Red flags**: Doesn't know either tool; confuses them.
**Green flags**: Explains partial encryption (SOPS) vs. full encryption (Sealed Secrets), mentions KMS integration.

### Q3: How does ArgoCD handle the SOPS-encrypted files in GitOps workflow?
**Why asked**: Tests practical GitOps implementation knowledge.
**Expected answer**: ArgoCD uses the `ksops` plugin (Kustomize + SOPS) or a custom config management plugin. During sync, ArgoCD's repo server decrypts the SOPS-encrypted files using credentials (AWS IAM role or KMS access) configured on the ArgoCD service account. The decrypted manifests are applied to the cluster but never stored in plaintext in Git or ArgoCD's cache. The ArgoCD repo server needs the decryption key access — for AWS KMS, the ArgoCD pod's ServiceAccount needs an IAM role with `kms:Decrypt` permission on the encryption key.
**Red flags**: Doesn't know how ArgoCD decrypts; says "the files are committed decrypted."
**Green flags**: Explains the ksops plugin, IAM role on ArgoCD's service account, mentions that plaintext never persists in Git.

### Q4: You used AWS Parameter Store over Secrets Manager. When would Secrets Manager be the better choice?
**Why asked**: Tests ability to evaluate trade-offs and make informed decisions.
**Expected answer**: Secrets Manager is better when: (1) You need automatic secret rotation (e.g., RDS database passwords with Lambda-based rotation). (2) You need cross-account secret sharing with resource policies. (3) You need versioning with staging labels (AWSCURRENT, AWSPREVIOUS). (4) Compliance requirements mandate rotation audit trails. Parameter Store is better for: static configuration, cost-sensitive workloads (free tier), hierarchical path-based organization, and when you don't need built-in rotation.
**Red flags**: Says "they're the same thing" or can't articulate when each is appropriate.
**Green flags**: Mentions automatic rotation as the key differentiator, discusses cost and hierarchical paths.

### Q5: Walk me through what happens when a new pod starts and needs secrets via ESO + IRSA.
**Why asked**: Tests end-to-end understanding of the secret delivery pipeline.
**Expected answer**: (1) Pod is scheduled with a ServiceAccount annotated with an IAM role ARN. (2) The EKS pod identity webhook mutates the pod spec to inject AWS environment variables (`AWS_ROLE_ARN`, `AWS_WEB_IDENTITY_TOKEN_FILE`) and mount a projected ServiceAccount token. (3) The pod's AWS SDK uses these to call STS `AssumeRoleWithWebIdentity`, exchanging the OIDC token for temporary AWS credentials. (4) Meanwhile, ESO's controller has already synced the ExternalSecret → Kubernetes Secret (this happens independently of pod startup). (5) The pod mounts the Kubernetes Secret as env vars or files. (6) The pod starts with secrets available — no Vault sidecar, no init container delay.
**Red flags**: Thinks the pod directly calls AWS Parameter Store; doesn't understand the OIDC token exchange.
**Green flags**: Describes the OIDC → STS flow, explains that ESO sync is independent of pod lifecycle.

### Q6: How do you ensure dev/intg/prod parity while still allowing environment-specific overrides?
**Why asked**: Tests infrastructure-as-code practices and DRY principles.
**Expected answer**: Shared base configuration in `values.yaml` (application defaults, health check paths, port numbers). Environment-specific overrides in `values-{env}.yaml` (replica counts, resource limits, issuer domains, feature flags). ArgoCD ApplicationSets or Kustomize overlays select the correct override file per target cluster. The key principle: the override files should be minimal — only what genuinely differs between environments. CI/CD pipelines deploy the same container image across all environments; only configuration changes. Drift detection via ArgoCD ensures the deployed state matches Git.
**Red flags**: Maintains entirely separate configs per environment; says "we just change things in prod directly."
**Green flags**: Explains the base + overlay pattern, mentions minimal overrides, drift detection.

### Q7: What would you do differently if starting the ESO migration from scratch?
**Why asked**: Tests reflection and continuous improvement mindset.
**Expected answer**: (1) Start with a migration script that auto-generates ExternalSecret manifests from existing Vault paths — reduce manual work for the remaining 17 services. (2) Implement a canary migration pattern — run both Vault and ESO simultaneously for a service and compare the synced values before cutover. (3) Add Prometheus metrics on ESO sync status (`externalsecret_sync_status`) to the monitoring dashboards from day one. (4) Consider using ClusterSecretStore for shared secrets (TLS certs, shared API keys) instead of per-namespace SecretStores. (5) Implement automated testing that verifies secrets exist and are non-empty in each environment post-deployment.
**Red flags**: "Nothing, the migration was perfect."
**Green flags**: Mentions automation for the remaining services, canary migration, monitoring ESO sync health.

---

## Tags & Cross-References

### Related STAR Stories
- [JWT Authentication](jwt-authentication.md) — issuer validation uses environment-specific Helm values managed by this infra
- [Monitoring Overhaul](monitoring-overhaul.md) — ESO sync metrics should be added to monitoring
- [API Versioning Evolution](api-versioning-evolution.md) — per-version Istio config deployed through this pipeline

### Interview Question Categories This Covers
- Infrastructure: Kubernetes secrets, ESO, IRSA, SOPS
- Security: Least-privilege IAM, encryption at rest, secret rotation
- DevOps/GitOps: ArgoCD, Helm values management, drift detection
- Cloud (AWS): Parameter Store, KMS, IAM roles for service accounts

### Behavioral Dimensions Demonstrated
- **Migration leadership**: Pilot strategy with template for remaining services
- **Security-first**: Least-privilege IAM policies per service
- **Operational maturity**: Runbooks, migration checklists, rollback plans
- **Cost awareness**: Parameter Store (free) over Secrets Manager ($0.40/secret/month)
