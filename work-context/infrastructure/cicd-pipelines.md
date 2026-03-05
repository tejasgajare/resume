# CI/CD Pipelines

> **Tags**: `#interview-ready` `#cicd` `#github-actions` `#argocd` `#gitops` `#sonarqube` `#managed-ci`
> **Role**: Software Engineer — RAVE Platform, HPE GreenLake
> **Cross-references**: [kubernetes-helm.md](./kubernetes-helm.md), [secrets-management.md](./secrets-management.md), [aws-terraform.md](./aws-terraform.md)

---

## Overview

I manage and contribute to the CI/CD pipeline infrastructure for the RAVE platform. Our pipeline is built on GitHub Actions using HPE's Managed CI framework (`glcp/managed-ci-workflow`), with ArgoCD handling GitOps-based deployments to EKS clusters. The pipeline encompasses code quality checks (SonarQube), Go linting, unit testing, Docker image building, Helm chart packaging, and automated deployment through ArgoCD sync. I also manage the PR auto-approval workflows and the deployment configuration repo (`mcd-deploy-proj-rave`).

---

## GitHub Actions Pipeline Architecture

### Managed CI Framework

HPE's GreenLake Cloud Platform uses a centralized CI framework called Managed CI (`glcp/managed-ci-workflow`). Rather than each team writing CI pipelines from scratch, we consume reusable workflow templates and extend them with pre/post hooks. This ensures consistency across HPE teams while allowing customization.

Our workflow files in `rave-cloud/.github/workflows/`:

```
workflows/
├── managed-ci-pr.yaml              # Main PR pipeline orchestrator
├── managed-ci-build-golang.yaml     # Go build workflow
├── managed-ci-unit-test-golang.yaml # Go unit test workflow
├── managed-ci-lint-golang.yaml      # Go linting workflow
├── managed-ci-quality-check.yaml    # SonarQube quality gates
├── managed-ci-merge.yaml           # Post-merge pipeline
├── managed-ci-final.yaml           # Final status reporting
├── managed-ci-pr-ft.yaml           # Functional test pipeline
├── mci-pre-check.yaml              # Custom pre-check hooks
├── mci-pre-test.yaml               # Custom pre-test hooks
├── mci-pre-build.yaml              # Custom pre-build hooks
├── mci-post-test.yaml              # Custom post-test hooks
├── mci-post-build.yaml             # Custom post-build hooks
├── build-rave.yml                  # Legacy build workflow
└── pr-checks.yml                   # Legacy PR checks
```

### PR Pipeline Flow

When a PR is opened against `rave-cloud`, the `managed-ci-pr.yaml` orchestrates this sequence:

```
PR Opened/Updated
  │
  ├─→ mci-setup (Managed CI framework initialization)
  │     └─→ Downloads github-context artifact
  │
  ├─→ github-context (Parses MCI configuration)
  │     └─→ Determines which stages to run (lint, test, build)
  │     └─→ Computes smart_sha for incremental builds
  │
  ├─→ quality-check (SonarQube)
  │     ├─→ mci-pre-check (custom RAVE hooks)
  │     ├─→ mci-quality-check (SonarQube scan)
  │     └─→ mci-post-check (report results)
  │
  ├─→ lint (Go linting)
  │     ├─→ mci-pre-lint (custom hooks)
  │     ├─→ mci-lint-golang (golangci-lint)
  │     └─→ mci-post-lint
  │
  ├─→ unit-test (Go tests)
  │     ├─→ mci-pre-test (custom setup)
  │     ├─→ mci-unit-test-golang (go test)
  │     └─→ mci-post-test (coverage upload)
  │
  ├─→ build (Docker image build)
  │     ├─→ mci-pre-build (custom hooks)
  │     ├─→ mci-build-golang (Docker build + push to JFrog)
  │     └─→ mci-post-build (tag artifacts)
  │
  └─→ final (Status reporting)
        └─→ Aggregate all stage results → PR status check
```

### Concurrency Management

I configured concurrency groups to prevent redundant CI runs:

```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.event.pull_request.number || github.ref }}
  cancel-in-progress: true
```

This ensures that pushing a new commit to an open PR cancels any in-progress CI run for that PR, saving runner time and preventing stale results.

### Smart SHA / Incremental Builds

The Managed CI framework computes a `smart_sha` that identifies which services changed in a PR. Since `rave-cloud` is a monorepo with 20+ services, rebuilding everything for a one-service change would be wasteful. The smart SHA enables incremental builds—only changed services are built and tested.

I manage the MCI configuration that maps directory paths to services:

```yaml
# managed-ci-workflow-config
services:
  caserelay:
    path: caseRelay/
    language: golang
  crmrelay:
    path: crmRelay/
    language: golang
  wellnessproducer:
    path: wellnessProducer/
    language: golang
  # ... 20+ more services
```

---

## SonarQube Integration

### Quality Gates

SonarQube runs as part of the `quality-check` stage in every PR. I configured:
- **Coverage threshold**: Minimum test coverage required for new code
- **Duplicated lines**: Maximum allowable code duplication
- **Code smells**: Tracked but non-blocking
- **Security hotspots**: Must be reviewed before merge

The SonarQube scan runs via the Managed CI `mci-quality-check.yaml` reusable workflow, which:
1. Checks out the code at the PR's head SHA
2. Runs `sonar-scanner` with the project configuration
3. Waits for the SonarQube quality gate result
4. Reports pass/fail back to the PR status check

### My SonarQube Contributions

I've worked on:
- Configuring the SonarQube project for `rave-cloud` with the correct source directories
- Setting up exclusion patterns for generated code, test fixtures, and vendor directories
- Resolving quality gate failures by addressing critical code smells and security hotspots
- Advocating for coverage improvements across the team

---

## Docker Image Build & Registry

### Build Pipeline

The Go services are built into Docker images and pushed to JFrog Artifactory:

```
Registry: hpeartifacts-docker-rave-wellnessautomation.jfrog.io
Image path: rave_handlers/{service}:{tag}
```

Image tags follow the pattern: `{branch}-{short-sha}` (e.g., `cloudmaster-ccp-bddfa5517`).

The build process:
1. **Multi-service build**: The `build-rave.yml` or Managed CI build stage iterates over changed services
2. **Docker build**: Each service has a Dockerfile in its directory
3. **Push to JFrog**: Authenticated via `regcred` secret
4. **Tag in metadata**: The ArgoCD deployment repo is updated with the new image tag

### Image Pull Configuration

In Kubernetes, pods pull images using the `regcred` imagePullSecret:

```yaml
spec:
  imagePullSecrets:
    - name: {{ .Values.global.dockerSecretName }}  # "regcred"
```

This secret contains JFrog Artifactory credentials, managed via SOPS/ESO.

---

## ArgoCD GitOps Deployment

### Architecture

ArgoCD is the deployment engine for RAVE. It watches the `mcd-deploy-proj-rave` repo and syncs Helm releases to the target EKS clusters.

```
mcd-deploy-proj-rave/
├── applications/
│   ├── rave/
│   │   ├── dev/
│   │   │   └── rave-dev-us-west-2/
│   │   │       └── rave/
│   │   │           ├── metadata.yaml          # ArgoCD app config
│   │   │           └── helm-values/
│   │   │               └── values-dev.yaml    # Environment values
│   │   ├── stage/
│   │   │   └── rave-stag-us-west-2/
│   │   │       └── rave/
│   │   │           ├── metadata.yaml
│   │   │           └── helm-values/
│   │   │               └── values-stage.yaml
│   │   └── prod/
│   │       └── rave-prod-us-west-2/
│   │           └── rave/
│   │               ├── metadata.yaml
│   │               └── helm-values/
│   │                   └── values-prod.yaml
│   └── wellness-proxy/
│       └── pr-auto-approve.yaml
└── README.md
```

### Deployment Flow

```
1. Developer merges PR to rave-cloud main branch
2. CI pipeline builds Docker images, pushes to JFrog
3. CI updates version in mcd-deploy-proj-rave/metadata.yaml
   (e.g., version: 1.0.1-cloudmaster-ccp-bddfa5517)
4. PR opened against mcd-deploy-proj-rave
5. PR auto-approved (for dev/stage) or manually reviewed (prod)
6. ArgoCD detects new commit in mcd-deploy-proj-rave
7. ArgoCD renders Helm chart with environment values
8. ArgoCD applies diff to target EKS cluster
9. Rolling update replaces pods with new image
```

### PR Auto-Approve

I configured PR auto-approval for non-production deployments:

```yaml
# applications/rave/pr-auto-approve.yaml
# Auto-approves deployment PRs for dev/stage environments
```

This accelerates the dev/stage deployment cycle—a merged code change reaches dev within minutes without manual intervention. Production deployments require explicit team approval.

### Metadata Configuration

The `metadata.yaml` defines the ArgoCD Application:

```yaml
cluster: https://EC5B6C9E7628A2779ECDADBB1543F54F.gr7.us-west-2.eks.amazonaws.com
github-group-names:
  teams:
    - rave
notification_email: peng.chen@hpe.com
helm-repo-url: https://hpeartifacts.jfrog.io/artifactory/api/helm/helm-harmony
chart: helm-rave
version: 1.0.1-cloudmaster-ccp-bddfa5517
helm-values-path: applications/rave/dev/rave-dev-us-west-2/rave/helm-values
namespace: rave
app-name: rave-d4e0ef
```

Key aspects:
- **chart + version**: Points to the exact Helm chart in JFrog
- **helm-values-path**: References the environment-specific values
- **github-group-names**: RBAC — only the `rave` team can approve changes
- **app-name**: Unique identifier with a hash suffix for collision avoidance

---

## Managed CI Workflow Configuration

### Custom Hooks

I manage the `managed-ci-workflow-config` repo, which defines RAVE-specific extensions to the Managed CI framework:

**Pre-check hooks** (`mci-pre-check.yaml`):
- Custom validation before SonarQube scan
- Ensure required labels are present on the PR

**Pre-test hooks** (`mci-pre-test.yaml`):
- Set up test dependencies (mock servers, test databases)
- Configure test environment variables

**Post-build hooks** (`mci-post-build.yaml`):
- Update deployment metadata with new image tags
- Trigger downstream workflows (deployment repo PR creation)

### Configuration

The MCI configuration for RAVE specifies:
- **Language**: Go (determines which build/test/lint workflows to use)
- **Build targets**: Which services to build based on changed paths
- **Test configuration**: Coverage thresholds, test timeouts
- **Quality gates**: SonarQube project key and quality profiles

---

## Environment Promotion Strategy

### Dev → Stage → Prod

```
dev:   Auto-deploy on merge to main
         ↓ (validation period: automated smoke tests)
stage: PR with version bump → auto-approved
         ↓ (validation period: integration tests + QA)
prod:  PR with version bump → manual team approval required
```

### Rollback Procedure

If a production deployment causes issues:
1. **Revert the deployment PR** in `mcd-deploy-proj-rave` — reverts the `version` in `metadata.yaml`
2. ArgoCD detects the revert and syncs the previous Helm chart version
3. Pods roll back to the previous image tag
4. Total rollback time: ~3-5 minutes

Because ArgoCD is purely declarative (desired state in Git), rollback is just a Git revert. No manual `kubectl` commands needed.

---

## CI Pipeline Optimizations I've Made

### 1. Concurrency Group Tuning
Configured `cancel-in-progress: true` to avoid wasted CI minutes on stale PR pushes.

### 2. Incremental Build Path Mapping
Mapped each service directory to the MCI service definition so only changed services rebuild. This reduced average PR CI time from ~20 minutes to ~8 minutes for single-service changes.

### 3. Docker Layer Caching
Ensured Dockerfiles are structured to maximize layer reuse—Go module downloads and base image layers are cached, only the final binary copy changes per build.

### 4. Parallel Job Execution
The MCI framework runs lint, test, and quality-check in parallel (not sequentially), shaving minutes off the PR feedback loop.

---

## Interview Deep-Dives

### Q: Explain your CI/CD pipeline end-to-end.

When I push code to a PR in our monorepo, GitHub Actions triggers the Managed CI framework. It first parses which services changed using path mapping, then runs SonarQube quality checks, Go linting, and unit tests in parallel. If all pass, it builds Docker images for changed services and pushes to JFrog Artifactory. A post-build hook creates a PR against our ArgoCD deployment repo with the new image tag. For dev/stage, this PR is auto-approved and ArgoCD syncs within minutes. For prod, the team reviews and approves. ArgoCD renders the Helm chart with environment-specific values and performs a rolling update. Rollback is a Git revert.

### Q: How do you handle secrets in CI/CD?

CI pipeline secrets (JFrog credentials, SonarQube tokens, GitHub tokens) are managed via GitHub Actions secrets at the org level and inherited via `secrets: inherit` in reusable workflows. Application secrets (MongoDB credentials, API keys) are separate—they're in SOPS-encrypted files or AWS Parameter Store, synced by ESO at runtime. The CI pipeline never handles application secrets directly.

### Q: What's the difference between your approach and a traditional CI/CD?

Traditional CI/CD might use Jenkins with imperative deploy scripts. Our approach is GitOps—the desired state is always in Git. ArgoCD continuously reconciles the cluster to match. Benefits: auditable history (every deployment is a Git commit), easy rollback (Git revert), drift detection (ArgoCD alerts if someone `kubectl apply`s manually). The trade-off is complexity—maintaining the deployment repo structure and ArgoCD configuration adds overhead.

### Q: How would you improve the pipeline?

I'd add canary deployments using Argo Rollouts instead of standard rolling updates. Currently, a bad deployment affects all pods simultaneously. With canary, we'd route 5% of traffic to the new version first, validate metrics, then gradually increase. I'd also add automated integration tests in the stage environment before promoting to prod—right now, QA is partially manual.

---

## Key Repos

| Repo | Purpose | My Role |
|------|---------|---------|
| `rave-cloud` | Service code + CI workflows | PR pipeline, MCI hooks |
| `mcd-deploy-proj-rave` | ArgoCD deployment config | Helm values, version management |
| `managed-ci-workflow-config` | MCI framework config | RAVE service mapping |
| `glcp/managed-ci-workflow` | Reusable CI workflows | Consumer (upstream) |

---

## Metrics & Impact

- **~8 min** average PR CI time (down from ~20 min via incremental builds)
- **3 environments** (dev/stage/prod) with automated promotion pipeline
- **Auto-deploy to dev** within minutes of merge
- **GitOps rollback** in ~3-5 minutes via Git revert
- **SonarQube quality gates** enforced on every PR
- **20+ services** built from a single monorepo with incremental CI
