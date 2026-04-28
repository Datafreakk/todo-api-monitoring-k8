# Platform Pipeline Templates — Azure Container Apps

Maintained by: **Platform Engineering**
Version: `v1.0.0`

Standardised CI/CD pipeline for services deployed to Azure Container Apps. Development teams extend this template — they configure what to deploy, the template handles how it deploys, how it rolls back, and how it validates.

---

## Repository Structure

```
pipeline-templates/
├── pipelines/
│   └── aca-service.yml              ← Teams extend this (the entry point)
├── templates/
│   ├── stages/
│   │   ├── build.yml                ← CI: build, scan, push
│   │   └── deploy.yml               ← CD: approve, deploy, flip, smoke test, rollback
│   └── steps/
│       └── trivy-scan.yml           ← Reusable Trivy vulnerability scan steps
├── examples/
│   └── azure-pipelines.yml          ← Copy this to your service repo
└── README.md
```

---

## Pipeline Flow

```
┌─────────────────────────────────────────────────────────────────────────┐
│ Trigger: push to main / release/*                                        │
└────────────────────────────┬────────────────────────────────────────────┘
                             │
                    ┌────────▼────────┐
                    │   CI_Build      │
                    │                 │
                    │  checkout       │
                    │  docker build   │
                    │  trivy scan ────┼── FAIL if CRITICAL/HIGH CVEs
                    │  push to ACR    │   (only on main/release)
                    └────────┬────────┘
                             │ succeeded()
          ┌──────────────────▼──────────────────┐
          │          Deploy_dev                  │
          │                                      │
          │  [Gate]      ADO Environment check   │
          │  [Deploy]    new revision @ 0% traffic│
          │              wait for Running state   │
          │  [Flip]      100% traffic → new rev  │
          │  [Smoke]     HTTP /health → 200?      │
          │  [Rollback]  ← only if smoke fails   │
          └──────────────────┬──────────────────┘
                             │ succeeded()
          ┌──────────────────▼──────────────────┐
          │         Deploy_staging               │  ← same flow, approval optional
          └──────────────────┬──────────────────┘
                             │ succeeded()
          ┌──────────────────▼──────────────────┐
          │         Deploy_production            │  ← approval required (ADO Environment)
          └─────────────────────────────────────┘
```

---

## Onboarding a New Service

### Step 1 — Platform team: one-time setup per service

**a) Create ADO Environments**

In ADO: `Settings > Pipelines > Environments` — create one environment per target:
- `dev`
- `staging`
- `production`

For `staging` and `production`, add approval checks:
- `Approvals and checks > Approvals` — add required approvers
- `Approvals and checks > Branch control` — restrict to `main` branch for production

**b) Create ADO Service Connections**

Both must use **Workload Identity Federation** — no client secrets stored:

| Connection | Role required |
|---|---|
| `sc-acr-prod` | `AcrPush` on the ACR |
| `sc-azure-platform` | `Contributor` on the target resource groups |

In ADO: `Settings > Service connections > New > Azure Resource Manager > Workload Identity Federation`

**c) Attach ACR to ACA**

The ACA managed identity needs `AcrPull` on the ACR:
```bash
# Get ACA managed identity
IDENTITY=$(az containerapp show \
  --name my-service-prod \
  --resource-group rg-myapp-prod \
  --query "identity.principalId" \
  --output tsv)

# Assign AcrPull
az role assignment create \
  --assignee "$IDENTITY" \
  --role AcrPull \
  --scope $(az acr show --name mycompanyacr --query id --output tsv)
```

**d) Create ACA apps (first time only)**

ACA apps must exist before the pipeline runs. Create them once with your base configuration:
```bash
az containerapp create \
  --name my-service-dev \
  --resource-group rg-myapp-dev \
  --environment your-aca-environment \
  --image mcr.microsoft.com/azuredocs/containerapps-helloworld:latest \
  --ingress external \
  --target-port 8080
```

The pipeline will update the image on every deploy — the initial image doesn't matter.

---

### Step 2 — Service team: add pipeline to your repo

Copy [examples/azure-pipelines.yml](examples/azure-pipelines.yml) to your repo root and update the parameters:

```yaml
resources:
  repositories:
    - repository: pipeline-templates
      type: git
      name: platform/pipeline-templates   # org/repo in ADO
      ref: refs/tags/v1.0.0               # ALWAYS pin to a release tag

extends:
  template: pipelines/aca-service.yml@pipeline-templates
  parameters:
    serviceName: my-service              # becomes the Docker image name
    acrName: mycompanyacr
    acrServiceConnection: sc-acr-prod
    azureServiceConnection: sc-azure-platform
    environments:
      - name: dev
        containerAppName: my-service-dev
        resourceGroup: rg-myapp-dev
        dependsOn: CI_Build
        healthCheckPath: /health
      - name: staging
        containerAppName: my-service-staging
        resourceGroup: rg-myapp-staging
        dependsOn: Deploy_dev
        healthCheckPath: /health
      - name: production
        containerAppName: my-service-prod
        resourceGroup: rg-myapp-prod
        dependsOn: Deploy_staging
        healthCheckPath: /health
```

Create the pipeline in ADO pointing at your `azure-pipelines.yml`. Done.

---

## Template Parameters Reference

### `pipelines/aca-service.yml`

| Parameter | Required | Default | Description |
|---|---|---|---|
| `serviceName` | ✅ | — | Service name — used as the Docker image name in ACR |
| `acrName` | ✅ | — | ACR name without `.azurecr.io` |
| `acrServiceConnection` | ✅ | — | ADO service connection with AcrPush on the ACR |
| `azureServiceConnection` | ✅ | — | ADO service connection with Contributor on resource groups |
| `environments` | ✅ | — | List of environments (see below) |
| `dockerfilePath` | ❌ | `Dockerfile` | Path to Dockerfile relative to repo root |
| `dockerContext` | ❌ | `.` | Docker build context |
| `buildArgs` | ❌ | `''` | Space-separated `KEY=VALUE` Docker build args |
| `trivySeverity` | ❌ | `CRITICAL,HIGH` | Severity levels that fail the build |

### Environment object fields

| Field | Required | Default | Description |
|---|---|---|---|
| `name` | ✅ | — | ADO Environment name — must exist in ADO |
| `containerAppName` | ✅ | — | Azure Container App resource name |
| `resourceGroup` | ✅ | — | Resource group of the Container App |
| `dependsOn` | ✅ | `CI_Build` | Stage name to depend on for sequencing |
| `healthCheckPath` | ❌ | `/health` | HTTP path for smoke test — must return 200 |

---

## Key Design Decisions

### Immutable image tags
Every image is tagged with the full 40-character Git commit SHA. The same tag is used from CI through to production — never rebuilt. This guarantees what was tested in staging is exactly what runs in production.

### Blue-green via ACA revisions
ACA's multiple-revision mode is the blue-green mechanism. The new revision is created at 0% traffic, verified (Running state + smoke test), then traffic is flipped atomically. The old revision stays warm for instant rollback.

### Automatic rollback
If the smoke test fails after a traffic flip, the pipeline automatically restores 100% traffic to the previous revision and deactivates the failed one. The pipeline exits red — a silent rollback that shows green would hide production incidents.

### PR builds vs merge builds
CI runs on every PR (build + scan only). Push to ACR only happens on `main` or `release/*` branches. This prevents PR builds from polluting the registry.

### Workload Identity Federation
All service connections use Workload Identity Federation — no client secrets stored in ADO or anywhere. Azure AD issues short-lived tokens per pipeline run. If you see service connections using a client secret, raise a platform ticket to migrate them.

### Approval gates via ADO Environments
Approval logic lives on the ADO Environment — not in YAML. This means:
- Approval configuration can be changed without a PR to the pipeline
- Approval history is tracked in ADO's audit log
- The same environment can be reused across services with consistent policy

---

## Updating the Template (Platform Team)

1. Make changes in a feature branch
2. Test against a non-production service pipeline
3. Merge to main via PR with at minimum one reviewer
4. Tag the release: `git tag v1.x.x && git push --tags`
5. Communicate the new tag and changelog to service teams via your platform newsletter/channel
6. Service teams update `ref: refs/tags/v1.x.x` in their repos at their own pace

**Do NOT make breaking changes without a major version bump and migration guide.**

---

## Troubleshooting

**Revision stuck in `Provisioning` state**
- Check ACA health probe configuration — if the probe path is wrong, the revision never becomes Running
- Check container startup logs: `az containerapp logs show --name <app> --resource-group <rg> --revision <rev> --tail 100`

**Smoke test failing (HTTP 000)**
- The health check URL may require auth — update `healthCheckPath` to an unauthenticated endpoint
- Check if ACA ingress is configured as `external` — internal ingress won't be reachable from the pipeline agent

**Push to ACR fails with 401**
- Verify the service connection has `AcrPush` on the correct ACR
- Verify the service connection uses Workload Identity Federation (not an expired client secret)

**Rollback job shows "No previous revision" error**
- This happens on the very first deployment — there is no previous revision to roll back to
- Manually set traffic to 0% for the failed revision and investigate container logs

**Pipeline fails on branch check (won't deploy)**
- CD stages only run on `main` and `release/*` branches by design
- Feature branch builds run CI only — this is intentional

---

## Health Check Contract

Your service **must** expose a health endpoint that:
- Returns HTTP 200 when the service is ready to handle traffic
- Is accessible without authentication
- Responds within 10 seconds
- Does not require a request body

Standard path is `/health`. If your framework uses a different path (e.g. `/healthz`, `/actuator/health`), set `healthCheckPath` in your environment config.
