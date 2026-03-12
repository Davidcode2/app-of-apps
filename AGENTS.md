# AGENTS.md - App of Apps Repository

This repository defines all applications running on Jakob's k3s cluster using the GitOps pattern with ArgoCD.

## 🎯 Philosophy

**GitOps All The Way.** Git is the source of truth. Commit changes here → ArgoCD syncs them to the cluster automatically.

**App-of-Apps Pattern.** One ArgoCD application (`main-gitops-app`) watches this repo and creates all other applications.

**Self-Healing & Auto-Pruning.** Apps sync automatically. Deleted manifests = deleted resources. Drift is corrected automatically.

## 📦 Repository Structure

```
app-of-apps/
├── app-of-apps.yml                      # The root ArgoCD app (creates all others)
│
├── Infrastructure Apps (Helm charts)
├── cert-manager.yaml                    # Let's Encrypt certificate manager
├── ingress-nginx-app.yaml               # Nginx ingress controller
├── hetzner-ccm-app.yaml                 # Hetzner Cloud Controller Manager
├── external-secrets-operator-app.yaml   # External Secrets Operator
├── argocd-infra-app.yaml                # ArgoCD ingress
│
├── Infrastructure Config (Plain manifests)
├── cluster-issuer-app.yaml              # cert-manager cluster issuer config
├── external-secrets-config-app.yaml     # External secrets config
│   └── external-secrets/
│       ├── aws-secret-store.yaml        # AWS Parameter Store connection
│       ├── aws-credentials-secret.yaml  # AWS credentials (manual)
│       └── *-external-secret.yaml       # ExternalSecret resources
│
├── Application Apps (Plain manifests)
├── blog-app.yaml
│   └── blog/
│       ├── blog-app-deployment.yaml
│       ├── blog-ingress-resource.yaml
│       └── blog-service.yaml
├── immoly-app.yaml
│   └── immoly/                          # Deployment, service, ingress, PVC, etc.
├── schluesselmomente-app.yaml
│   └── schluesselmomente/
├── joy-alemazung-app.yaml
│   └── joy-alemazung/
├── joy-alemazung-cms-app.yaml
│   └── joy-alemazung-cms/
├── umami-app.yaml
│   └── umami/
└── uptime-kuma-app.yaml
    └── uptime-kuma/
```

## 🏗️ Application Types

### 1. Helm Chart Applications (Infrastructure)

External Helm charts for standard components:

**Pattern:**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: ingress-nginx
  namespace: argocd
spec:
  source:
    repoURL: https://kubernetes.github.io/ingress-nginx
    chart: ingress-nginx
    targetRevision: "4.10.0"
    helm:
      values: |
        # Custom values here
```

**Examples:**
- `cert-manager` - Certificate management
- `ingress-nginx` - Ingress controller
- `hetzner-ccm` - Hetzner load balancer integration
- `external-secrets-operator` - Secrets sync from AWS

### 2. Plain Manifest Applications

Custom applications with plain YAML manifests in subdirectories:

**Pattern:**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: immoly
  namespace: argocd
spec:
  source:
    repoURL: https://github.com/davidcode2/app-of-apps
    targetRevision: HEAD
    path: immoly  # Folder in this repo
  destination:
    namespace: immoly
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

**Examples:**
- `blog`, `immoly`, `schluesselmomente` - Web applications
- `umami` - Analytics platform
- `uptime-kuma` - Uptime monitoring

## 🔑 Naming Conventions

### Application Names
- Lowercase, hyphenated
- Match the project/service name
- Example: `joy-alemazung-cms`

### File Names
Root-level app definitions: `<name>-app.yaml`
- `blog-app.yaml` → creates ArgoCD app named `blog`
- `immoly-app.yaml` → creates ArgoCD app named `immoly`

### Folder Names
Match the application name exactly:
- `blog-app.yaml` → `blog/` folder
- `immoly-app.yaml` → `immoly/` folder

### Manifest Names (inside folders)
Descriptive, with resource type suffix:
- `<app>-deployment.yaml`
- `<app>-service.yaml`
- `<app>-ingress-resource.yaml`
- `<app>-namespace.yaml`
- `<app>-persistentvolumeclaim.yaml`

## 🔄 Sync Policies

### Standard Sync Policy (Most Apps)
```yaml
syncPolicy:
  automated:
    prune: true       # Delete resources removed from Git
    selfHeal: true    # Revert manual changes
  syncOptions:
    - CreateNamespace=true  # Auto-create target namespace
```

### When to Disable Auto-Sync
- Critical infrastructure (you might want manual approval)
- Apps with external state that shouldn't auto-rollback
- Testing/development apps

## 🗂️ Application Folder Structure

### Simple Application
```
blog/
├── blog-app-deployment.yaml
├── blog-service.yaml
└── blog-ingress-resource.yaml
```

### Complex Application (with database)
```
immoly/
├── immoly-namespace.yaml
├── immoly-app-deployment.yaml
├── immoly-app-service.yaml
├── immoly-app-ingress-resource.yaml
├── immoly-postgres-deployment.yaml
├── immoly-postgres-service.yaml
├── immoly-volume-persistentvolumeclaim.yaml
├── db-secret.yaml                      # Placeholder (real secrets via ExternalSecret)
├── env-configmap.yaml
└── immoly-migration-job.yaml
```

## 🔐 Secrets Management

**Never commit real secrets to Git.**

### Pattern: External Secrets Operator

1. **Store secret in AWS Parameter Store:**
   ```bash
   aws ssm put-parameter \
     --name /k8s/immoly/db-password \
     --value "secret-value" \
     --type SecureString
   ```

2. **Create ExternalSecret in `external-secrets/` folder:**
   ```yaml
   apiVersion: external-secrets.io/v1beta1
   kind: ExternalSecret
   metadata:
     name: immoly-db
     namespace: immoly
   spec:
     secretStoreRef:
       name: aws-parameter-store
       kind: ClusterSecretStore
     target:
       name: immoly-db-secret
     data:
       - secretKey: password
         remoteRef:
           key: /k8s/immoly/db-password
   ```

3. **Reference secret in deployment:**
   ```yaml
   env:
     - name: DB_PASSWORD
       valueFrom:
         secretKeyRef:
           name: immoly-db-secret
           key: password
   ```

### Manual Secrets (AWS Credentials)
The AWS credentials secret must be created manually:
```bash
kubectl create secret generic aws-credentials \
  --from-literal=access-key-id=<key> \
  --from-literal=secret-access-key=<secret> \
  -n external-secrets
```

See `external-secrets/SETUP.md` for details.

## ✨ Adding a New Application

### 1. Create Application Folder
```bash
mkdir my-app
```

### 2. Create Kubernetes Manifests
```bash
cd my-app
# Create deployment, service, ingress, etc.
```

### 3. Create ArgoCD App Definition
```bash
# In repo root
cat > my-app-app.yaml <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
  finalizers:
  - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/davidcode2/app-of-apps
    targetRevision: HEAD
    path: my-app
  destination:
    server: https://kubernetes.default.svc
    namespace: my-app
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
EOF
```

### 4. Commit and Push
```bash
git add my-app/ my-app-app.yaml
git commit -m "Add my-app application"
git push
```

### 5. Watch ArgoCD Sync
ArgoCD detects the change within ~3 minutes and deploys automatically.

```bash
kubectl get applications -n argocd
argocd app get my-app
```

## 🌐 Ingress Pattern

All public applications use this ingress pattern:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app-ingress
  namespace: my-app
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - my-app.jakob-lingel.dev
      secretName: my-app-tls
  rules:
    - host: my-app.jakob-lingel.dev
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: my-app-service
                port:
                  number: 80
```

**cert-manager** automatically provisions Let's Encrypt certificates.

## 🏭 Infrastructure Dependencies

Applications depend on infrastructure apps being healthy:

**Dependency Order:**
1. `hetzner-ccm` - Cloud Controller Manager (for LoadBalancer services)
2. `cert-manager` - Certificate management
3. `ingress-nginx` - Ingress controller
4. `external-secrets-operator` - Secrets sync
5. `cluster-issuer` - Let's Encrypt issuer config
6. `external-secrets-config` - AWS SecretStore config
7. **Application apps** - Can now deploy

ArgoCD handles this automatically via sync waves (if configured) or natural dependency resolution.

## 🔍 Debugging

### Check App Status
```bash
# List all apps
kubectl get applications -n argocd

# Detailed app status
argocd app get <app-name>

# Sync status and health
argocd app list
```

### Force Sync
```bash
argocd app sync <app-name>
```

### View Sync Errors
```bash
kubectl describe application <app-name> -n argocd
```

### Manual Rollback
```bash
argocd app rollback <app-name> <revision>
```

## 🚨 Common Issues

### App OutOfSync but Healthy
- Check if manual changes were made in cluster
- Auto-sync will fix it (if enabled)
- Or: `argocd app sync <app-name>`

### App Degraded
- Check pod status: `kubectl get pods -n <namespace>`
- Check logs: `kubectl logs <pod> -n <namespace>`
- Check events: `kubectl get events -n <namespace>`

### External Secret Not Syncing
- Verify AWS Parameter Store has the value
- Check External Secrets logs: `kubectl logs -n external-secrets -l app.kubernetes.io/name=external-secrets`
- Verify AWS credentials secret exists

### Ingress Not Working
- Check ingress resource: `kubectl get ingress -A`
- Check cert-manager: `kubectl get certificate -A`
- Check nginx logs: `kubectl logs -n ingress-nginx -l app.kubernetes.io/component=controller`

## 📊 Current Applications

### Production Applications
- **blog** - Jakob's personal blog (Ghost)
- **immoly** - Real estate calculation tool
- **schluesselmomente** - Client website (frontend + backend)
- **schluesselmomente-cms** - Strapi CMS for Schlüsselmomente (admin.schluesselmomente-freiburg.de)
- **joy-alemazung** - Client website (Ghost)
- **joy-alemazung-cms** - Strapi CMS for Joy Alemazung
- **umami** - Web analytics
- **uptime-kuma** - Uptime monitoring

### Infrastructure
- **ingress-nginx** - Ingress controller (Hetzner LB)
- **cert-manager** - Let's Encrypt certificates
- **external-secrets** - Syncs secrets from AWS
- **hetzner-ccm** - Hetzner Cloud integration

## 🔄 Workflow Best Practices

### Making Changes
1. Create feature branch
2. Update manifests
3. Test locally (if possible): `kubectl apply --dry-run=client -f <file>`
4. Commit and push
5. Open PR
6. Merge → ArgoCD auto-deploys

### Monitoring Changes
1. Watch ArgoCD UI or CLI
2. Check application health
3. Monitor logs for errors
4. Verify endpoints are accessible

### Rollback Strategy
Git is the source of truth. To rollback:
1. Revert the Git commit
2. ArgoCD syncs the previous state
3. Or use ArgoCD rollback: `argocd app rollback <app> <revision>`

---

## 🔗 Related Repositories

- **infra** - Terraform + Ansible for cluster provisioning
- Individual application source code repos

## 📝 Maintenance Notes

**When files are added or deleted, update this section:**

### Current File Structure
```
.
├── app-of-apps.yml
├── argocd-infra-app.yaml
├── argocd-ingress
│   └── argocd-ingress.yaml
├── blog-app.yaml
│   └── blog/
├── cert-manager.yaml
├── cluster-issuer-app.yaml
│   └── cluster-issuer/
├── external-secrets-config-app.yaml
│   └── external-secrets/
├── external-secrets-operator-app.yaml
├── hetzner-ccm-app.yaml
├── immoly-app.yaml
│   └── immoly/
├── ingress-nginx-app.yaml
├── jakob-lingel-app.yaml
│   └── jakob-lingel/
├── joy-alemazung-app.yaml
│   └── joy-alemazung/
├── joy-alemazung-cms-app.yaml
│   └── joy-alemazung-cms/
├── schluesselmomente-app.yaml
│   └── schluesselmomente/
├── umami-app.yaml
│   └── umami/
└── uptime-kuma-app.yaml
    └── uptime-kuma/
```

---

**Remember:** Every change to this repo is a deployment. Git commit = production change (within ~3 min).

<!-- BEGIN BEADS INTEGRATION -->
## Issue Tracking with bd (beads)

**IMPORTANT**: This project uses **bd (beads)** for ALL issue tracking. Do NOT use markdown TODOs, task lists, or other tracking methods.

### Why bd?

- Dependency-aware: Track blockers and relationships between issues
- Git-friendly: Dolt-powered version control with native sync
- Agent-optimized: JSON output, ready work detection, discovered-from links
- Prevents duplicate tracking systems and confusion

### Quick Start

**Check for ready work:**

```bash
bd ready --json
```

**Create new issues:**

```bash
bd create "Issue title" --description="Detailed context" -t bug|feature|task -p 0-4 --json
bd create "Issue title" --description="What this issue is about" -p 1 --deps discovered-from:bd-123 --json
```

**Claim and update:**

```bash
bd update <id> --claim --json
bd update bd-42 --priority 1 --json
```

**Complete work:**

```bash
bd close bd-42 --reason "Completed" --json
```

### Issue Types

- `bug` - Something broken
- `feature` - New functionality
- `task` - Work item (tests, docs, refactoring)
- `epic` - Large feature with subtasks
- `chore` - Maintenance (dependencies, tooling)

### Priorities

- `0` - Critical (security, data loss, broken builds)
- `1` - High (major features, important bugs)
- `2` - Medium (default, nice-to-have)
- `3` - Low (polish, optimization)
- `4` - Backlog (future ideas)

### Workflow for AI Agents

1. **Check ready work**: `bd ready` shows unblocked issues
2. **Claim your task atomically**: `bd update <id> --claim`
3. **Work on it**: Implement, test, document
4. **Discover new work?** Create linked issue:
   - `bd create "Found bug" --description="Details about what was found" -p 1 --deps discovered-from:<parent-id>`
5. **Complete**: `bd close <id> --reason "Done"`

### Auto-Sync

bd automatically syncs via Dolt:

- Each write auto-commits to Dolt history
- Use `bd dolt push`/`bd dolt pull` for remote sync
- No manual export/import needed!

### Important Rules

- ✅ Use bd for ALL task tracking
- ✅ Always use `--json` flag for programmatic use
- ✅ Link discovered work with `discovered-from` dependencies
- ✅ Check `bd ready` before asking "what should I work on?"
- ❌ Do NOT create markdown TODO lists
- ❌ Do NOT use external issue trackers
- ❌ Do NOT duplicate tracking systems

For more details, see README.md and docs/QUICKSTART.md.

## Landing the Plane (Session Completion)

**When ending a work session**, you MUST complete ALL steps below. Work is NOT complete until `git push` succeeds.

**MANDATORY WORKFLOW:**

1. **File issues for remaining work** - Create issues for anything that needs follow-up
2. **Run quality gates** (if code changed) - Tests, linters, builds
3. **Update issue status** - Close finished work, update in-progress items
4. **PUSH TO REMOTE** - This is MANDATORY:
   ```bash
   git pull --rebase
   bd dolt push
   git push
   git status  # MUST show "up to date with origin"
   ```
5. **Clean up** - Clear stashes, prune remote branches
6. **Verify** - All changes committed AND pushed
7. **Hand off** - Provide context for next session

**CRITICAL RULES:**
- Work is NOT complete until `git push` succeeds
- NEVER stop before pushing - that leaves work stranded locally
- NEVER say "ready to push when you are" - YOU must push
- If push fails, resolve and retry until it succeeds

<!-- END BEADS INTEGRATION -->
