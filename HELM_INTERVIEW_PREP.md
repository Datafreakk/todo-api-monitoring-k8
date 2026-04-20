# Helm Interview Prep Notes

---

## 1. HELM FUNDAMENTALS

### What is Helm?
Helm is the package manager for Kubernetes. It packages Kubernetes manifests (YAML files) and their configuration into reusable, versioned units called **Charts**. Helm simplifies deployment, upgrades, and rollbacks of applications on Kubernetes without manual YAML management.

### Core Problem Helm Solves
- **Repetitive YAML**: Writing same manifests for dev, staging, prod differs only in values (image, replicas, etc.)
- **No versioning**: No built-in versioning for Kubernetes applications
- **No dependency management**: Hard to manage inter-dependencies between services
- **No upgrade/rollback**: Manual kubectl apply is error-prone for updates

### Three Core Concepts

**Chart**: A packaged Kubernetes application
- Contains templates (YAML blueprints with variables)
- Contains default values (image, replicas, ports, etc.)
- Versioned (v1.0.0, v1.1.0)
- Deployable to any Kubernetes cluster

**Values**: Configuration variables for charts
- Default values in `values.yaml`
- Override via `--set` flag or custom value files
- Enables same chart for different environments

**Release**: A specific deployment instance of a chart
```
$ helm install my-release mychart  # Creates release "my-release" from chart "mychart"
$ helm install prod-release mychart --values prod-values.yaml  # Different config
```

Same chart, different releases = different deployments with different configurations.

## 2. CHART STRUCTURE

### Directory Layout
```
mychart/
├── Chart.yaml              # Metadata (name, version, description)
├── values.yaml             # Default configuration values
├── templates/
│   ├── deployment.yaml     # Kubernetes Deployment manifest template
│   ├── service.yaml        # Service manifest template
│   ├── configmap.yaml      # ConfigMap if needed
│   ├── _helpers.tpl        # Helper template functions (reusable snippets)
│   └── NOTES.txt           # Post-install instructions shown to user
├── Chart.lock              # Dependency lock file (like package-lock.json)
└── charts/                 # Dependency charts (other Charts this depends on)
```

### Chart.yaml Anatomy
```yaml
apiVersion: v2                    # Helm 3+ format
name: todo-api                    # Chart name
description: Todo API Chart       # Description
type: application                 # application | library
version: 1.2.3                    # Chart version (not app version)
appVersion: "2.1.0"               # Application version inside chart
home: https://github.com/...      # Project homepage
sources:                          # Source code repositories
  - https://github.com/path
maintainers:
  - name: Praveen
    email: praveen@example.com
dependencies:                     # Dependent charts
  - name: postgresql
    version: "12.0.0"
    repository: "https://charts.bitnami.com"
```

### values.yaml
Default configuration applied to all templates:
```yaml
replicaCount: 3

image:
  registry: docker.io
  repository: myapp/todo-api
  tag: "1.2.3"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80
  targetPort: 8080

ingress:
  enabled: false
  class: "nginx"
  hosts:
    - host: api.example.com
      paths:
        - path: /
          pathType: Prefix

resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 250m
    memory: 256Mi

autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 80

nodeSelector: {}
tolerations: []
affinity: {}
```

## 3. HELM TEMPLATING

### Template Syntax
Helm uses Go templating language with Sprig functions:

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "mychart.fullname" . }}
  labels:
    {{- include "mychart.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "mychart.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "mychart.selectorLabels" . | nindent 8 }}
    spec:
      containers:
      - name: {{ .Chart.Name }}
        image: "{{ .Values.image.registry }}/{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        ports:
        - name: http
          containerPort: {{ .Values.service.targetPort }}
        resources:
          {{- toYaml .Values.resources | nindent 10 }}
        {{- if .Values.env }}
        env:
          {{- range $key, $value := .Values.env }}
          - name: {{ $key }}
            value: {{ quote $value }}
          {{- end }}
        {{- end }}
```

### Template Functions
```yaml
{{ .Values.replicaCount }}           # Access value
{{ include "chart.name" . }}         # Call helper template
{{ .Chart.Name }}                    # Chart metadata
{{ .Release.Name }}                  # Release name passed at install
{{ .Namespace }}                     # K8s namespace
{{ quote "string" }}                 # Add quotes around string
{{ toYaml .Values.resources }}       # Convert to YAML
{{ nindent 10 . }}                   # Indent YAML by 10 spaces
{{ if .Values.enabled }}...{{ end }} # Conditional rendering
{{ range .Values.items }}...{{ end}} # Loop/iteration
```

### Conditional Rendering
```yaml
# Install only if enabled
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
...
{{- end }}

# Different logic based on environment
{{- if eq .Values.environment "production" }}
  replicas: 10
{{- else if eq .Values.environment "staging" }}
  replicas: 3
{{- else }}
  replicas: 1
{{- end }}
```

## 4. HELM COMMANDS

### Install & Upgrade
```bash
# Install chart from repository
helm install my-release mychart                    # Default values
helm install my-release mychart --values prod-values.yaml
helm install my-release mychart --set replicaCount=5

# Upgrade existing release
helm upgrade my-release mychart                    # Update to latest version
helm upgrade my-release mychart --values new-values.yaml
helm upgrade --install my-release mychart          # Install if not exists, upgrade if exists

# Rollback to previous release
helm rollback my-release                           # Rollback to previous version
helm rollback my-release 1                         # Rollback to specific revision
```

### List & Inspect
```bash
helm list                                          # List all releases
helm list -n production                            # List releases in namespace
helm status my-release                             # Show release status
helm get values my-release                         # Show values used in release
helm get manifest my-release                       # Show rendered templates
helm history my-release                            # Show release history (revisions)
```

### Template Debugging
```bash
# Render templates without installing
helm template my-release mychart                   # Show rendered YAML
helm template my-release mychart --values vals.yaml

# Validate templates
helm lint mychart                                  # Validate chart syntax and values
```

### Repository Management
```bash
helm repo add bitnami https://charts.bitnami.com  # Add chart repository
helm repo list                                     # List configured repositories
helm repo update                                   # Fetch latest charts from repos
helm search repo postgres                          # Search for chart in repos
```

### Uninstall
```bash
helm uninstall my-release                         # Delete release
helm uninstall my-release --keep-history          # Keep history for rollback
```

## 5. HELM WORKFLOW PATTERNS

### Environment Management
```
# Single chart, multiple environments
helm install prod-api my-app --values values-prod.yaml
helm install staging-api my-app --values values-staging.yaml
helm install dev-api my-app --values values-dev.yaml
```

### GitOps Integration (ArgoCD)
```yaml
# ArgoCD Application manifest
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: todo-api
spec:
  source:
    repoURL: https://github.com/company/helm-charts
    path: charts/todo-api
    helm:
      values: |
        replicaCount: 3
        image:
          tag: v1.2.3
  destination:
    server: https://kubernetes.default.svc
    namespace: prod
```

### Dependency Management
```yaml
# Chart.yaml dependencies
dependencies:
  - name: postgresql
    version: "12.0.0"
    repository: "https://charts.bitnami.com"
    condition: postgresql.enabled

  - name: redis
    version: "17.0.0"
    repository: "https://charts.bitnami.com"
    condition: redis.enabled
```

```bash
# Update dependencies
helm dependency update mychart  # Downloads charts/postgresql and charts/redis
```

### Secrets Management
```bash
# Pass secrets at install time
helm install my-release mychart \
  --set-string database.password="secret123" \
  --set-string api.key="api-key-xyz"

# Better: Use external secret management
helm install my-release mychart \
  -f <(kubectl get secret db-creds -o jsonpath='{.data.values}' | base64 -d)
```

## 6. COMMON PATTERNS & BEST PRACTICES

### Organizing Values
```yaml
# Group related configs
image:
  registry: docker.io
  repository: myapp
  tag: "1.0.0"

service:
  type: ClusterIP
  port: 80

ingress:
  enabled: false
  hosts: []

resources:
  requests: {}
  limits: {}

autoscaling:
  enabled: false
  minReplicas: 1
  maxReplicas: 3
```

### Naming Conventions
- Chart names: lowercase, dashes (e.g., `todo-api`)
- Release names: lowercase, dashes (e.g., `prod-api`)
- Helper templates: prefix with underscore (e.g., `_helpers.tpl`)
- Template names: match resource kind (e.g., `deployment.yaml`, `service.yaml`)

### Helper Templates
```yaml
# templates/_helpers.tpl
{{- define "mychart.name" -}}
{{ .Chart.Name }}
{{- end }}

{{- define "mychart.fullname" -}}
{{ .Release.Name }}-{{ .Chart.Name }}
{{- end }}

{{- define "mychart.labels" -}}
helm.sh/chart: {{ include "mychart.chart" . }}
app.kubernetes.io/name: {{ include "mychart.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}
```

### File Structure Best Practices
```
myapp-chart/
├── Chart.yaml                 # Always required
├── values.yaml                # Provide sensible defaults
├── values-schema.json         # (V3.6+) Validate values
├── templates/
│   ├── NOTES.txt             # User-friendly post-install
│   ├── _helpers.tpl          # Reusable template functions
│   ├── deployment.yaml       # Main workload
│   ├── service.yaml          # Service exposure
│   ├── configmap.yaml        # Configuration
│   └── ingress.yaml          # External access
├── charts/                   # Dependencies (auto-managed)
├── README.md                 # Documentation
└── LICENSE                   # License file
```

---

## 7. INTERVIEW QUESTIONS & ANSWERS

### Q: "What is Helm and why would you use it over plain Kubernetes manifests?"

**A**: Helm is Kubernetes's package manager that packages YAML manifests into versioned, reusable Charts. Key advantages:

1. **Template Variables**: A single chart works for dev/staging/prod by overriding values (image tag, replicas) without modifying YAML
2. **Versioning**: Charts are versioned (v1.0.0) enabling reproducible deployments
3. **Dependency Management**: Charts can depend on other charts (PostgreSQL, Redis) avoiding duplicated manifests
4. **Release Management**: Track deployments with `helm install`, `helm upgrade`, `helm rollback` instead of manual kubectl apply
5. **Reusability**: Share charts across teams/projects (public chart repositories like Bitnami)
6. **Rollback**: One command to revert to previous configuration if deployment breaks

Without Helm, you'd manually create/update YAML files, manage versions manually, and struggle with configuration differences between environments.

### Q: "Explain the difference between Chart, Values, and Release"

**A**: Three distinct concepts:

- **Chart**: The blueprint/package containing templates and default values. Immutable once created. Example: `mychart:v1.2.0`
- **Values**: Configuration variables that customize the chart. Can be default (values.yaml) or override at install time. Example: `replicaCount: 5`
- **Release**: A specific deployment instance combining chart + values. Each release is named and versioned independently. Example: `helm install prod-api mychart --set replicaCount=10` creates a release named "prod-api"

Analogy: Chart = cookbook recipe, Values = ingredients you choose, Release = actual meal you cook and eat.

### Q: "How would you manage configuration differences between dev, staging, and production?"

**A**: Use single chart with environment-specific values files:

```bash
# Single chart structure
my-app/
  Chart.yaml
  values.yaml              # Defaults
  values-dev.yaml          # Dev overrides
  values-staging.yaml      # Staging overrides
  values-prod.yaml         # Prod overrides
  templates/

# Install with environment-specific values
helm install dev-app my-app --values values-dev.yaml
helm install staging-app my-app --values values-staging.yaml
helm install prod-app my-app --values values-prod.yaml

# Or merge multiple values files
helm install prod-app my-app \
  --values values.yaml \
  --values values-prod.yaml \
  --values secrets-prod.yaml
```

**Alternative**: Use Kustomize or ArgoCD for layering, but Helm values files are simpler for most cases.

### Q: "What's the difference between `helm install` and `helm upgrade`?"

**A**: 
- **helm install**: Creates a new release from a chart. Fails if release name already exists
- **helm upgrade**: Updates an existing release to a new/same chart version. Fails if release doesn't exist

**In Practice**:
```bash
helm install demo mychart                    # Creates release "demo"
helm install demo mychart                    # ERROR: release "demo" already exists

helm upgrade demo mychart                    # Updates "demo" release
helm upgrade demo mychart --values new.yaml  # Updates with new values

helm upgrade --install demo mychart          # Install OR upgrade (common pattern)
```

**Key Difference**: `install` creates, `upgrade` modifies existing. `upgrade --install` handles both automatically.

### Q: "How would you rollback a bad Helm deployment?"

**A**: Helm maintains release history and enables one-command rollbacks:

```bash
# View release history
helm history my-release
# Shows all revisions with timestamps and status

# Rollback to previous revision
helm rollback my-release                # Rollback to revision before current

# Rollback to specific revision
helm rollback my-release 2               # Rollback to revision 2

# After rollback, verify
helm status my-release
helm get values my-release
```

**How it Works**: Each `helm install`/`helm upgrade` creates a revision. Rollback re-applies the previous revision's values and manifests without changing release name.

**Example**:
```
Revision 1: app v1.0.0 (initial install)
Revision 2: app v1.0.1 (upgrade - broken)
Revision 3: app v1.0.0 (rollback - back to working state)
```

### Q: "How do you pass secrets to Helm charts?"

**A**: Multiple approaches with different security tradeoffs:

**Option 1: Command-line flag (Simple, insecure)**
```bash
helm install my-app mychart --set-string database.password="secret123"
# Problem: Password in bash history, visible in process list
```

**Option 2: Values file (Better, but file must be secured)**
```yaml
# secrets.yaml (gitignored, never committed)
database:
  password: secret123
  user: admin
```
```bash
helm install my-app mychart -f secrets.yaml
# Problem: Still need to protect secrets.yaml file
```

**Option 3: Kubernetes Secret First (Recommended)**
```bash
kubectl create secret generic db-creds \
  --from-literal=password=secret123 \
  --from-literal=user=admin

# Chart reference secret
helm install my-app mychart \
  --set-string mysql.existingSecret=db-creds
```

**Option 4: External Secret Management (Production)**
```bash
# Use HashiCorp Vault, AWS Secrets Manager, etc.
# Retrieve at deploy time and pass to Helm
SECRET=$(vault kv get -field=password secret/db)
helm install my-app mychart --set-string database.password="$SECRET"
```

**Best Practice**: Use Kubernetes Secrets + external secret operator (ES0), or Sealed Secrets for encryption at rest.

### Q: "What are Helm Dependencies and how do you manage them?"

**A**: Dependencies are charts packaged inside your chart (required services like PostgreSQL, Redis).

**Declaration** (Chart.yaml):
```yaml
dependencies:
  - name: postgresql
    version: "12.0.0"
    repository: "https://charts.bitnami.com/bitnami"
    condition: postgresql.enabled  # Only install if this value is true
    alias: db                        # Reference as {{ .Values.db }} instead of postgresql
    tags:
      - backend-services           # Install if --set tags.backend-services=true
```

**Management**:
```bash
# Download dependencies to charts/ subdirectory
helm dependency update mychart

# Lists dependencies
helm dependency list mychart

# Clean dependencies
helm dependency build mychart
```

**Usage** (values.yaml):
```yaml
postgresql:
  enabled: true
  auth:
    username: postgres
    password: secret123
  primary:
    persistence:
      size: 10Gi

redis:
  enabled: false  # Don't install Redis
```

**Benefits**: 
- Avoid duplicating PostgreSQL manifests across charts
- Version PostgreSQL independently from your app
- Share dependency versions across multiple charts

### Q: "What's a Helm hook and when would you use it?"

**A**: Hooks are templates that run at specific points in release lifecycle (install, upgrade, delete).

**Common Hooks**:
```yaml
# pre-install: Run before chart installation
# post-install: Run after installation
# pre-upgrade: Run before upgrade
# post-upgrade: Run after upgrade
# pre-delete: Run before deletion
# post-delete: Run after deletion
# pre-rollback: Run before rollback
# post-rollback: Run after rollback
```

**Example: Database Migration Hook**
```yaml
# templates/_migration-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ include "mychart.fullname" . }}-migrate
  annotations:
    "helm.sh/hook": pre-install,pre-upgrade
    "helm.sh/hook-weight": "-5"
    "helm.sh/hook-delete-policy": before-hook-creation
spec:
  template:
    spec:
      containers:
      - name: migrate
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        command: ["./migrate.sh"]
      restartPolicy: Never
```

**Use Cases**:
- Database migrations before app upgrade
- Cleanup tasks before deletion
- Health checks after install
- Configuration updates before rollback

### Q: "How would you lint and validate a Helm chart?"

**A**: Helm provides commands to catch errors before deployment:

```bash
# Validate chart syntax and best practices
helm lint mychart
# Checks:
# - Chart.yaml is valid YAML
# - Template syntax errors
# - Values validation if values-schema.json exists
# - Missing required fields

# Render templates without installing (catch rendering errors)
helm template mychart
# Shows generated Kubernetes YAML before installation

# With custom values
helm template mychart --values values-prod.yaml

# Validate rendered YAML against Kubernetes API
helm template mychart | kubectl apply --dry-run=client -f -
# Fails if manifests are invalid for your K8s version

# More thorough validation (requires kubeconform or kubeval)
helm template mychart | kubeconform -
```

**Best Practice**: Add to CI/CD pipeline:
```bash
# Check syntax
helm lint charts/*

# Check rendering
helm template mychart > /dev/null

# Validate against schema
helm template mychart --values values.yaml | kubeval
```

### Q: "What are Helm values schema and why are they important?"

**A**: JSON schema file (values-schema.json) that validates user-provided values before installation.

**Example**:
```json
{
  "$schema": "https://json-schema.org/draft-07/schema#",
  "type": "object",
  "properties": {
    "replicaCount": {
      "type": "integer",
      "minimum": 1,
      "maximum": 100,
      "description": "Number of pod replicas"
    },
    "image": {
      "type": "object",
      "properties": {
        "repository": {
          "type": "string",
          "pattern": "^[a-z0-9-./]+$"
        },
        "tag": {
          "type": "string",
          "pattern": "^[a-zA-Z0-9._-]+$"
        }
      },
      "required": ["repository", "tag"]
    },
    "service": {
      "type": "object",
      "properties": {
        "port": {
          "type": "integer",
          "minimum": 1,
          "maximum": 65535
        }
      }
    }
  },
  "required": ["replicaCount", "image", "service"]
}
```

**Validation**:
```bash
# Helm 3.7+ validates during install
helm install my-app mychart --values invalid-values.yaml
# ERROR: values.service.port must be integer between 1-65535

# Manual validation
helm lint mychart
```

**Benefits**:
- Catch configuration errors early
- Prevent invalid deployments
- Document expected types and ranges
- Enable IDE validation

### Q: "How would you integrate Helm with CI/CD (e.g., GitHub Actions, GitLab CI)?"

**A**: Example GitHub Actions workflow:

```yaml
# .github/workflows/deploy.yml
name: Deploy Helm Chart

on:
  push:
    branches: [main]
  
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Helm
        uses: azure/setup-helm@v3
        with:
          version: '3.12.0'
      
      - name: Lint Chart
        run: helm lint ./charts/myapp
      
      - name: Template Rendering
        run: |
          helm template myapp ./charts/myapp \
            --values values-prod.yaml \
            --values secrets.yaml > rendered.yaml
          cat rendered.yaml
      
      - name: Validate Against Kubernetes
        run: |
          helm template myapp ./charts/myapp | \
          kubeval --strict
      
      - name: Deploy to Kubernetes
        run: |
          helm upgrade --install myapp ./charts/myapp \
            --namespace production \
            --values values-prod.yaml \
            --values secrets.yaml \
            --wait \
            --timeout 5m
      
      - name: Rollback on Failure
        if: failure()
        run: helm rollback myapp -n production
```

**Best Practices**:
1. Lint before deploy (catch errors early)
2. Template rendering test (validate syntax)
3. Dry-run install (`--dry-run=server`)
4. Deploy with `--wait` flag
5. Automated rollback on failure

### Q: "Compare Helm vs Kustomize for Kubernetes templating"

**A**: Both solve templating problem but with different approaches:

| Aspect | Helm | Kustomize |
|--------|------|-----------|
| **What is it** | Package manager + templating | Templating + overlays |
| **Language** | Go templating | YAML patching |
| **Learning curve** | Moderate (Go syntax) | Low (YAML-based) |
| **Versioning** | Built-in (Chart.yaml) | None (external versioning) |
| **Dependency mgmt** | Yes (Chart.yaml) | No (manual) |
| **Reusability** | High (charts are packages) | Medium (bases + overlays) |
| **Distribution** | Hub repositories | Git-based |
| **Use case** | Published applications | Custom deployments |

**Helm**: Ideal for ISVs, ready-made charts (databases, monitoring)
**Kustomize**: Ideal for internal, Git-OpSec-friendly customizations

**Can use together**: Helm + Kustomize (Helm generates, Kustomize patches)

### Q: "What's the difference between Helm 2 and Helm 3?"

**A**: Helm 3 introduced major breaking changes:

| Feature | Helm 2 | Helm 3 |
|---------|--------|--------|
| **Tiller** | Required server component (security risk) | Removed (client-only) |
| **Installation** | `helm init` + Tiller setup | Direct install, no setup |
| **Permissions** | Tiller had global cluster access | Uses kubectl permissions |
| **Release storage** | Tiller secrets | Kubernetes Secrets |
| **Chart.yaml** | apiVersion: v1 | apiVersion: v2 |
| **Dependencies** | Use `requirements.yaml` | Use `Chart.yaml` |
| **Stability** | Less stable | Stable, boring release |

**Security**: Helm 3 is more secure (no Tiller cluster-admin), so never use Helm 2 in production.

### Q: "How would you troubleshoot a failed Helm installation?"

**A**: Systematic debugging approach:

```bash
# 1. Check release status
helm status my-release
# Shows current state and last operation

# 2. View rendered templates
helm get manifest my-release
# Compare what Helm tried to deploy vs. what you expected

# 3. Check Kubernetes events
kubectl events --for=release/my-release
# Shows Kubernetes errors (image pull failures, etc.)

# 4. Check pod logs
kubectl logs -n default deployment/my-release
# App-level debugging

# 5. Describe failing resources
kubectl describe pod my-release-xxxx
# Shows detailed error messages

# 6. Dry-run to test templates
helm template my-release mychart --values values.yaml
# Verify templates render correctly before install

# 7. Get values used in release
helm get values my-release
# Verify correct values were applied

# 8. Check for syntax errors
helm lint mychart
# Validates chart structure
```

**Common Issues**:
- Image not found (wrong registry/tag)
- PersistentVolumeClaim not bound (infrastructure issue)
- Service port conflicts (port already in use)
- RBAC permissions (ServiceAccount lacks permissions)
- Network policies blocking traffic (Ingress/Network Policy)

### Q: "How do you handle configuration drift in Helm releases?"

**A**: Configuration drift = deployed state differs from Helm values.

**Detection**:
```bash
# Compare current release to Helm values
helm get manifest my-release > current.yaml
helm template my-release mychart --values values.yaml > expected.yaml
diff current.yaml expected.yaml

# Or use GitOps tools (ArgoCD, Flux) for automatic detection
```

**Prevention**:
1. **Never kubectl apply/edit**: Always use `helm upgrade` for changes
2. **GitOps**: Store values in Git, deploy from Git
3. **Auditing**: Enable Kubernetes audit logs for manual changes
4. **Reconciliation**: Regular `helm upgrade --dry-run` to detect drift
5. **Locking**: Use RBAC to prevent direct kubectl edits

**Example - GitOps with ArgoCD**:
```yaml
# argocd-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
spec:
  source:
    repoURL: https://github.com/company/infra
    path: helm/values
    helm:
      releaseName: my-app
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true  # Delete resources removed from Git
      selfHeal: true  # Resync if drift detected
```

ArgoCD automatically detects and corrects drift.

---

## SUMMARY TABLE

| Concept | Purpose | Key Command |
|---------|---------|-------------|
| **Chart** | Packaged app template | `helm create`, `helm lint` |
| **Values** | Configuration variables | `--values`, `--set` |
| **Release** | Deploymentinstance | `helm install`, `helm upgrade` |
| **Template** | YAML with variables | Go templating in `templates/` |
| **Hook** | Lifecycle automation | `helm.sh/hook` annotation |
| **Dependency** | Required sub-chart | `Chart.yaml dependencies` |

---

**Last Updated**: April 2026 | **Focus**: Kubernetes Deployment & Interview Preparation
