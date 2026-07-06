# ESS Community Deploy Plan

## Strategy: Mac local first → server
1. ~~Mac local (k3d) — smoke test, validate config, learn the stack~~ ✅ **DONE**
2. Server — real deploy with external DB, real domain, Let's Encrypt

See `mac-local.md` for Step 1 detail.

### Mac local — completed 2026-07-06
- k3d cluster `ess-helm` running (k8s v1.34.2, no audit logging — see `work/k3d-simple.yaml`)
- cert-manager v1.20.3 installed manually (setup script incompatible with Mac — audit log kills informer sync)
- Self-signed CA at `~/.config/ess-helm-ca/ca.crt` — trusted in System keychain
- ESS Community 26.6.3-dev deployed via `charts/matrix-stack`
- Users: `alkesh` / `Admin@12345`, `raj` / `Admin@12345`
- Element Web: `https://element.ess.localhost` ✅ login + chat verified
- **Key gotcha:** `setup_test_cluster.sh` audit policy (`level: Metadata`) causes cert-manager webhook deadlock on Mac — use `work/k3d-simple.yaml` instead

---

## What
Deploy Element Server Suite Community via Helm (`matrix-stack` chart) onto Kubernetes.
Chart source: `oci://ghcr.io/element-hq/ess-helm/matrix-stack`

## Stack components
- Synapse (Matrix homeserver)
- Matrix Authentication Service (MAS) — OIDC auth
- HAProxy — routing + .well-known
- PostgreSQL — bundled or external
- Element Web — chat client
- Element Admin — admin console
- Matrix RTC Backend — video calls
- Redis — Synapse worker pub/sub

---

## Phase 1: Decisions (fill before touching infra)

| Decision | Options | Notes |
|---|---|---|
| Server name | `<your-domain.tld>` | **Cannot change later without DB reset** |
| K8s target | K3s (new VPS) / existing cluster / cloud K8s | K3s = simplest single-node |
| TLS | Let's Encrypt / cert files (wildcard) / external reverse proxy | LE = easiest |
| PostgreSQL | Bundled (chart-managed) / external | External = production-safe |
| Node size | ≥2 CPU, ≥2 GB RAM | Minimum for K3s quick setup |

---

## Phase 2: Infrastructure

### 2a — DNS records (all point to server IP)
```
<server-name.tld>          A  <SERVER_IP>   # server name + .well-known
matrix.<server-name.tld>   A  <SERVER_IP>   # Synapse
account.<server-name.tld>  A  <SERVER_IP>   # MAS
mrtc.<server-name.tld>     A  <SERVER_IP>   # RTC backend
chat.<server-name.tld>     A  <SERVER_IP>   # Element Web
admin.<server-name.tld>    A  <SERVER_IP>   # Element Admin
```

### 2b — Kubernetes (K3s single-node)
```bash
curl -sfL https://get.k3s.io | sh -
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
```

### 2c — Helm
```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

### 2d — cert-manager (for Let's Encrypt)
```bash
helm repo add jetstack https://charts.jetstack.io
helm upgrade --install cert-manager jetstack/cert-manager \
  --namespace cert-manager --create-namespace \
  --set crds.enabled=true
```

---

## Phase 3: Values files (stored in `~/ess-config-values/`)

### 3a — hostnames.yaml
```bash
curl -L https://raw.githubusercontent.com/element-hq/ess-helm/refs/heads/main/charts/matrix-stack/ci/fragments/quick-setup-hostnames.yaml \
  -o ~/ess-config-values/hostnames.yaml
# Edit hostnames in file to match your DNS above
```

### 3b — tls.yaml (Let's Encrypt)
- Configure `clusterIssuer` pointing to LE prod/staging
- Reference: `docs/advanced.md` TLS section

### 3c — postgresql.yaml (external DB only)
- Set `postgresql.enabled: false`
- Provide external connection string
- Reference: `docs/advanced.md` DB section

---

## Phase 4: Install

```bash
# Namespace
kubectl create namespace ess

# Helm install (adjust -f flags per your choices)
helm upgrade --install --namespace ess ess \
  oci://ghcr.io/element-hq/ess-helm/matrix-stack \
  -f ~/ess-config-values/hostnames.yaml \
  -f ~/ess-config-values/tls.yaml \
  # -f ~/ess-config-values/postgresql.yaml  (if external DB)
  --wait
```

---

## Phase 5: Post-install

### Create first admin user
```bash
kubectl exec -n ess -it deploy/ess-matrix-authentication-service \
  -- mas-cli manage register-user
```

### Verify
1. Open `https://chat.<server-name.tld>` → login
2. Check federation: https://federationtester.matrix.org/
3. Login from Element X mobile

---

## Phase 6: Production hardening (after smoke test)

- [ ] External PostgreSQL (remove bundled, take DB backups)
- [ ] S3 media storage (Synapse media PVC is not HA)
- [ ] SMTP → enables user self-registration via MAS
- [ ] Synapse workers (offload traffic, scale beyond ~100 users)
- [ ] Monitoring: ServiceMonitor + Prometheus Operator
- [ ] Backup plan: DB dumps + secret recovery (`ess-generated` secret)

---

## Key files reference

| File | Purpose |
|---|---|
| `charts/matrix-stack/values.yaml` | Full default values (generated, read-only) |
| `charts/matrix-stack/values.schema.json` | Schema for validation |
| `charts/matrix-stack/source/` | Edit here, not values.yaml directly |
| `docs/advanced.md` | External DB, TLS, MAS SMTP, workers |
| `docs/maintenance.md` | Upgrade, backup, restore |
| `docs/troubleshooting.md` | Common failure modes |

---

## Open questions
- [ ] Server name decided?
- [ ] Target infra (VPS provider / existing K8s)?
- [ ] TLS approach (LE / wildcard cert / external proxy)?
- [ ] External DB or bundled for first deploy?
