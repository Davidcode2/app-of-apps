# Paperless-NGX - Networking & Deployment Guide

## Overview

Paperless-NGX document management system deployed on a hybrid k3s cluster:
- **Data plane**: Runs exclusively on `acer-laptop` (homelab node with taints)
- **Networking**: Exposed via Tailscale Funnel (no public IP required)
- **DNS**: CNAME from `paperless.jakob-lingel.dev` → Tailscale Funnel URL

## Architecture

```
User (Internet)
    ↓
paperless.jakob-lingel.dev (CNAME)
    ↓
acer-arch.tailb781ce.ts.net (Tailscale Funnel)
    ↓ (WireGuard tunnel)
acer-laptop (100.115.249.19)
    ↓
localhost:8080 (hostPort)
    ↓
Paperless Web Pod
```

## Problem History

### Initial Challenge
Paperless runs on `acer-laptop` which:
- Is behind NAT/home network (no public IP)
- Cannot be reached by ingress-nginx on cloud nodes (separate network stacks)
- Has taints to prevent other workloads from scheduling there

### Approaches Considered

**❌ TCP Proxy (Failed)**
- Deployed nginx TCP proxies on cloud nodes
- Attempted to forward port 80 to acer-laptop:8080 via tailscale
- **Failed**: Port 80 already used by ingress-nginx on all cloud nodes
- Result: Pods stuck in Pending state

**❌ Cloudflare Tunnel (Rejected)**
- Would expose paperless directly to internet
- **Rejected**: Requires migrating entire `jakob-lingel.dev` domain to Cloudflare DNS
- User wanted to keep DNS at DigitalOcean for other subdomains

**❌ Tailscale Kubernetes Operator (Too Complex)**
- Would expose services via Tailscale Ingress
- **Issue**: Requires cloud nodes to route to acer-laptop's pod network (10.42.5.0/24)
- K3s CNI doesn't traverse tailscale, so cloud nodes couldn't reach the pod

**✅ Tailscale Funnel (Selected)**
- Runs directly on acer-laptop
- Exposes `localhost:8080` to internet via `https://acer-arch.tailb781ce.ts.net`
- Simple, works with existing tailscale setup
- No DNS migration required

## Current Implementation

### Pod Configuration
- Paperless web pod uses `hostPort: 8080` on acer-laptop
- Accessible locally at `http://127.0.0.1:8080`

### Tailscale Funnel Setup
```bash
# On acer-laptop
ssh jakob@100.115.249.19
tailscale funnel --bg 8080
```

### DNS Configuration
- **CNAME**: `paperless.jakob-lingel.dev` → `acer-arch.tailb781ce.ts.net`
- Managed in Terraform: `infra/terraform/global/dns/jakob-lingel.dev.tf`

### Important Limitation

**HTTPS only works on the Tailscale Funnel URL directly.**

Tailscale Funnel provides TLS certificates only for `*.tailb781ce.ts.net` domains. When accessing via the CNAME (`paperless.jakob-lingel.dev`), you'll get a certificate mismatch error:

```
An error occurred during a connection to paperless.jakob-lingel.dev.
PR_END_OF_FILE_ERROR

Error code: PR_END_OF_FILE_ERROR

The page you are trying to view cannot be shown because the authenticity 
of the received data could not be verified.
```

**Access URLs:**
- ✅ **Working**: `https://acer-arch.tailb781ce.ts.net`
- ❌ **Not working**: `https://paperless.jakob-lingel.dev` (certificate mismatch)

The CNAME is kept for consistency and potential future solutions, but use the Tailscale Funnel URL for access.

## Troubleshooting

### Symptom: Cannot access paperless

**Check 1: Verify Paperless pod is running**
```bash
kubectl get pods -n paperless-ngx
# Should see paperless-web pod Running on acer-laptop
```

**Check 2: Verify Tailscale connectivity**
```bash
ssh jakob@100.115.249.19
tailscale status
# Should show acer-laptop connected
```

**Check 3: Verify Funnel is active**
```bash
tailscale funnel status
```

Expected output:
```
https://acer-arch.tailb781ce.ts.net (Funnel on)
|-- / proxy http://127.0.0.1:8080
```

**Check 4: Test local connectivity**
```bash
curl http://127.0.0.1:8080
# Should return paperless HTML or redirect to login
```

**Check 5: Restart Funnel**
```bash
tailscale funnel --off
tailscale funnel --bg 8080
```

### Common Error: PR_END_OF_FILE_ERROR

**Cause**: Attempting to access `https://paperless.jakob-lingel.dev` instead of the Tailscale Funnel URL.

**Solution**: Use `https://acer-arch.tailb781ce.ts.net` directly.

### Common Error: Connection timeout

**Cause**: Acer-laptop powered off or tailscale disconnected.

**Solution**: 
1. Ensure acer-laptop is powered on and connected to internet
2. Check tailscale status on acer-laptop
3. Verify paperless pod is running

## Maintenance

### Service Availability
- Paperless is only available when acer-laptop is online
- If acer-laptop reboots, tailscale funnel auto-starts (runs as systemd service)
- Consider setting up monitoring alerts for acer-laptop connectivity

### Data Backup
- Paperless data stored in PVC on acer-laptop
- Ensure regular backups of `/usr/src/paperless/media` mount

### Updates
- To update paperless: modify image tag in `09-web.yaml`
- ArgoCD will automatically rollout new version
- Ensure acer-laptop has sufficient resources for updates

## Related Files

- `09-web.yaml` - Paperless web deployment (hostPort: 8080)
- `11-ingress.yaml` - Ingress resource (currently unused due to Tailscale Funnel)
- `01-namespace.yaml` - paperless-ngx namespace
- Infrastructure DNS: `../infra/terraform/global/dns/jakob-lingel.dev.tf`
