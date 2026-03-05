# AWS Infrastructure & Terraform

> **Tags**: `#interview-ready` `#infrastructure` `#aws` `#terraform` `#iam` `#eks` `#s3`
> **Role**: Software Engineer — RAVE Platform, HPE GreenLake
> **Cross-references**: [secrets-management.md](./secrets-management.md), [kubernetes-helm.md](./kubernetes-helm.md), [cicd-pipelines.md](./cicd-pipelines.md)

---

## Overview

I manage and contribute to the AWS infrastructure that underpins the RAVE (Remote Analysis Virtualization Engine) platform at HPE GreenLake. RAVE operates across three distinct AWS accounts—dev, intg/stage, and prod—each hosting EKS clusters, S3 buckets, IAM roles, Parameter Store secrets, and KMS keys. The infrastructure is provisioned via Terraform modules maintained by the CCP (Common Cloud Platform) team, with RAVE-specific configurations managed through environment-specific repos and Helm values.

My infrastructure work spans IAM role engineering for service-level AWS access, S3 bucket management for manual case attachments, KMS key administration for SOPS-encrypted secrets, and Parameter Store integration for the External Secrets Operator (ESO) migration. I work across the boundary between platform infrastructure (owned by CCP/SRE) and application-level infrastructure (owned by RAVE).

---

## AWS Account Architecture

### Multi-Account Strategy

RAVE operates in three isolated AWS accounts within HPE's organizational structure:

| Environment | AWS Account ID  | EKS Cluster                    | Region      | Purpose                        |
|-------------|-----------------|--------------------------------|-------------|--------------------------------|
| dev         | `590183756478`  | `rave-dev-us-west-2`           | us-west-2   | Development, feature testing   |
| stage       | *(separate)*    | `rave-stag-us-west-2`          | us-west-2   | Integration, pre-prod validation |
| prod        | `481824204946`  | `rave-prod-us-west-2`          | us-west-2   | Production workloads           |

Each account has its own VPC, EKS cluster, IAM policies, KMS keys, and Parameter Store hierarchy. This separation ensures blast radius containment—a misconfigured IAM policy in dev cannot affect prod resources.

### How I Work Across Accounts

I don't have direct AWS console access for most operations. Instead, I:
1. Define infrastructure-as-code in Terraform modules or Helm values
2. Submit PRs to the appropriate config repo (e.g., `aws-wellness_dev-config`, `aws-wellness_prod-config`)
3. CCP/SRE team reviews and applies Terraform changes
4. For Kubernetes-level resources (ServiceAccounts, RBAC), I deploy via ArgoCD through `mcd-deploy-proj-rave`

This workflow enforces separation of duties—I define *what* infrastructure is needed, and the platform team provisions it. For time-sensitive changes, I coordinate directly with CCP engineers.

---

## IAM Roles & Service Accounts

### IRSA (IAM Roles for Service Accounts)

I designed and implemented the IAM-to-Kubernetes binding for RAVE services using AWS IRSA (IAM Roles for Service Accounts). This is the mechanism that allows Kubernetes pods to assume AWS IAM roles without storing long-lived credentials.

**Architecture:**
```
Pod (serviceAccountName: manualcase-attachments-sa)
  → ServiceAccount annotation: eks.amazonaws.com/role-arn
    → IAM Role: rave-{env}-{region}-manualcase-attachments-iam-role
      → IAM Policy: s3:PutObject, s3:GetObject on manualcase-attachments bucket
```

**The ServiceAccount Template** (`manualcase-attachments-sa.yaml`):
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: {{ .Values.global.manualcases.s3.serviceAccount }}
  namespace: {{ .Values.global.namespace }}
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::{{ .Values.global.awsAccountId }}:role/rave-{{ .Values.global.environment }}-{{ .Values.global.manualcases.s3.region }}-manualcase-attachments-iam-role
```

**Values wiring** (from `values-dev.yaml`):
```yaml
global:
  awsAccountId: "590183756478"
  manualcases:
    s3:
      bucketSuffix: "manualcase-attachments"
      region: "us-west-2"
      serviceAccount: "manualcase-attachments-sa"
```

The pattern: each environment's `values-{env}.yaml` specifies the AWS account ID and S3 configuration. The Helm template constructs the full IAM role ARN dynamically. This means the same chart works across all three environments—only the values file changes.

### Custom ServiceAccount for ESO (GLCP-325064)

One of my key infrastructure contributions is the custom ServiceAccount work for the ESO migration on rave-prod (Jira: GLCP-325064). The default Kubernetes ServiceAccount doesn't have AWS IAM bindings, so services that need to read from AWS Parameter Store via ESO require a dedicated ServiceAccount with the appropriate IRSA annotation.

I created a pattern where:
1. A new ServiceAccount is created per service (or shared across related services)
2. The SA is annotated with an IAM role ARN that has `ssm:GetParameter` permissions on the RAVE Parameter Store path
3. The ESO `SecretStore` CRD references this ServiceAccount for authentication
4. Pods mount secrets via `ExternalSecret` CRDs that reference the `SecretStore`

This was critical for the prod migration because prod has stricter IAM policies—you can't use a blanket role; each service needs scoped permissions.

---

## S3 Bucket Management

### Manual Case Attachments

The RAVE caserelay service allows support engineers to attach files to cases. These attachments are stored in S3 buckets with the naming convention:

```
rave-{env}-{region}-manualcase-attachments
```

I configured:
- **Bucket creation**: Via Terraform in the `ccp-terraform-aws-s3` module
- **IAM policy**: Scoped to `s3:PutObject`, `s3:GetObject`, `s3:DeleteObject` on the specific bucket ARN
- **IRSA binding**: The caserelay pod uses `serviceAccountName: manualcase-attachments-sa` which assumes the IAM role with S3 permissions
- **Encryption**: Server-side encryption with AWS-managed keys (SSE-S3)

The caserelay deployment explicitly sets the service account:
```yaml
spec:
  serviceAccountName: {{ .Values.global.manualcases.s3.serviceAccount }}
```

This ensures only the caserelay pod (and any other pod using this SA) can access the attachments bucket. Other RAVE services use the `default` ServiceAccount and cannot reach S3.

---

## KMS Key Management

### SOPS Encryption Keys

Each environment has a dedicated AWS KMS key used for SOPS encryption of secrets in the GitOps repos (`rave-sops-dev`, `rave-sops-prod`, `rave-sops-stage`). The KMS key aliases follow the pattern:

```
arn:aws:kms:us-west-2:{account-id}:alias/eso/rave/rave-{env}/rave-{env}-secrets
```

For example, the dev SOPS file `mongodb-credentials.yaml` references:
```yaml
sops:
  kms:
    - arn: arn:aws:kms:us-west-2:590183756478:alias/eso/rave/rave-dev/rave-dev-secrets
```

I manage the SOPS configuration for these repos, including:
- Setting up `.sops.yaml` creation rules that auto-select the correct KMS key per environment
- Encrypting/decrypting secrets during rotation
- Ensuring the `encrypted_regex: ^(data)$` pattern only encrypts the `data` field, leaving metadata readable

### ESO Parameter Store KMS

The AWS Parameter Store parameters used by ESO are encrypted with separate KMS keys managed by the CCP team. I don't directly manage these keys, but I ensure the IAM roles attached to ESO ServiceAccounts have `kms:Decrypt` permissions on the relevant keys.

---

## Terraform Contributions

### Infrastructure Repos

I work with several Terraform-managed repos:

| Repo                           | Purpose                                    |
|--------------------------------|-------------------------------------------|
| `aws-wellness_dev-config`      | Dev account AWS resource definitions       |
| `aws-wellness_intg-config`     | Integration/stage account definitions      |
| `aws-wellness_prod-config`     | Production account definitions             |
| `ccp-terraform-aws-s3`         | Shared S3 bucket Terraform module          |
| `ccp-rave-*-cluster`           | EKS cluster configurations                 |

### My Terraform Workflow

I don't run `terraform apply` directly. My workflow:

1. **Identify need**: A new service requires an S3 bucket, IAM role, or Parameter Store path
2. **Write Terraform**: Add resource definitions or module invocations to the appropriate config repo
3. **PR with context**: Include the Jira ticket, architecture diagram, and IAM policy justification
4. **CCP review**: The platform team reviews for security, naming conventions, and blast radius
5. **Apply via pipeline**: Terraform runs in a controlled CI/CD pipeline with state locking

### Example: Adding a New Parameter Store Path for ESO

When migrating a service to ESO, I need a new Parameter Store path:
```
/eso/rave/rave-{env}/rave-{env}-secrets/rave/{service}/{key}
```

I add the parameter definition to the environment config repo with:
- The parameter path
- The encrypted value (encrypted with the environment's KMS key)
- Tags for team ownership and rotation tracking

---

## Network Architecture

### VPC & Connectivity

Each RAVE EKS cluster lives in a dedicated VPC with:
- **Private subnets** for EKS worker nodes (no direct internet access)
- **NAT Gateways** for outbound internet (pulling images, calling external APIs)
- **VPC Peering / PrivateLink** for MongoDB Atlas connectivity
- **Service mesh (Istio)** for intra-cluster traffic management

### MongoDB PrivateLink

RAVE services connect to MongoDB Atlas via AWS PrivateLink, avoiding public internet. The Istio `DestinationRule` and `ServiceEntry` configurations I manage ensure:
- DNS resolution for MongoDB private endpoints (e.g., `dev-rave-pl-0.ehjcd.mongodb.net`)
- TCP keepalive settings (300s time, 60s interval) to prevent connection drops
- TLS enforcement on port 27017

```yaml
apiVersion: networking.istio.io/v1
kind: ServiceEntry
metadata:
  name: mongo-private-endpoint
spec:
  hosts:
    - dev-rave-pl-0.ehjcd.mongodb.net
    - dev-rave-pl-1.ehjcd.mongodb.net
  location: MESH_EXTERNAL
  ports:
    - number: 27017
      name: mongodb
      protocol: TLS
  resolution: DNS
```

---

## Interview Deep-Dives

### Q: How do you manage multi-account AWS infrastructure?

We use separate AWS accounts per environment (dev/stage/prod) with Terraform-managed infrastructure. I define resource configurations in environment-specific repos, and the CCP platform team applies them. Each account has isolated VPCs, IAM policies, and KMS keys. The same Helm charts and Terraform modules work across all environments—only the variable files change (account IDs, endpoint URLs, replica counts). This separation ensures a dev misconfiguration can't impact production.

### Q: Explain IRSA and why you use it.

IRSA (IAM Roles for Service Accounts) lets Kubernetes pods assume AWS IAM roles without storing credentials. I create a ServiceAccount with an `eks.amazonaws.com/role-arn` annotation, which tells the EKS OIDC provider to inject temporary STS credentials into the pod. This is more secure than shared secrets because credentials are short-lived, automatically rotated, and scoped to specific pods. I use it for S3 access (caserelay attachments) and ESO Parameter Store reads.

### Q: What would you do differently with the AWS setup?

I'd push for more self-service Terraform. Currently, IAM and infrastructure changes require CCP team involvement, which creates bottlenecks. I'd advocate for Terraform modules with guardrails (OPA policies, Sentinel) that let application teams provision approved resource types independently. I'd also standardize the Parameter Store hierarchy earlier—our current paths evolved organically and could be more consistent.

### Q: How would this scale to 10x the number of services?

The IRSA pattern scales well—each service gets its own ServiceAccount and IAM role. The bottleneck would be Terraform state management and IAM policy limits. I'd introduce Terraform workspaces or Terragrunt for better state isolation, and consolidate IAM policies using wildcards where security allows. For Parameter Store, I'd implement a naming convention enforcer in CI to prevent path collisions.

---

## Key Repos

| Repo | Purpose | My Role |
|------|---------|---------|
| `aws-wellness_dev-config` | Dev AWS infra | Define IAM, S3, Parameter Store resources |
| `aws-wellness_prod-config` | Prod AWS infra | Production infrastructure definitions |
| `ccp-terraform-aws-s3` | S3 module | Consume for RAVE buckets |
| `ccp-rave-*-cluster` | EKS cluster config | Cluster-level requirements |
| `mcd-deploy-proj-rave` | ArgoCD deployment | Helm values with AWS references |
| `rave-sops-dev/prod/stage` | SOPS secrets | KMS-encrypted secret management |

---

## Metrics & Impact

- **3 AWS accounts** managed across dev/stage/prod
- **20+ microservices** with IRSA-enabled ServiceAccounts
- **ESO migration**: Eliminating HashiCorp Vault dependency, moving to native AWS Parameter Store
- **Zero credential leaks**: All secrets encrypted at rest (KMS) and in transit (TLS), no long-lived credentials in pods
- **Infrastructure-as-code coverage**: 100% of RAVE AWS resources defined in Terraform or Helm
