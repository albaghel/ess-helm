# Mac Local Deploy (k3d)

Deploy ESS Community on Mac using k3d (K3s inside Docker).
No real DNS. No real certs. Server name = `ess.localhost`.
Docker already installed. kubectl already installed. uv already installed.

---

## What you get
- All services at `*.ess.localhost` (self-signed CA, trusted once)
- Ingress bound to localhost:80 + localhost:443
- Bundled PostgreSQL
- No federation (local only)

URLs after deploy:
- Element Web: `https://element.ess.localhost`
- Element Admin: `https://admin.ess.localhost`
- Synapse: `https://synapse.ess.localhost`
- MAS: `https://mas.ess.localhost`
- RTC: `https://mrtc.ess.localhost`

---

## Step 1: Install missing tools

```bash
# Helm
brew install helm

# k3d (K3s in Docker)
brew install k3d
```

Verify:
```bash
helm version
k3d version
```

---

## Step 2: Install Python deps (uv)

From repo root:
```bash
cd /Users/Alkes/ess-helm/ess-helm
uv sync
source .venv/bin/activate
```

---

## Step 3: Setup test cluster

From repo root (Docker must be running):
```bash
./scripts/setup_test_cluster.sh
```

This script:
- Creates k3d cluster named `ess-helm`
- Installs Traefik ingress bound to localhost:80/443
- Installs cert-manager
- Creates self-signed CA at `~/.config/ess-helm-ca/`
- Creates namespace `ess` with wildcard cert for `*.ess.localhost`

Verify cluster up:
```bash
kubectl --context k3d-ess-helm get nodes
kubectl --context k3d-ess-helm -n ess get all
```

---

## Step 4: Trust the self-signed CA (do once)

```bash
# Add CA to macOS keychain
sudo security add-trusted-cert -d -r trustRoot \
  -k /Library/Keychains/System.keychain \
  ~/.config/ess-helm-ca/ca.crt
```

Browsers will trust `*.ess.localhost` after this. No browser warnings.

---

## Step 5: Deploy ESS

```bash
helm -n ess upgrade -i ess charts/matrix-stack \
  -f charts/matrix-stack/ci/test-cluster-mixin.yaml \
  -f charts/matrix-stack/ci/example-default-enabled-components-values.yaml \
  --kube-context k3d-ess-helm \
  --wait
```

Takes 2-5 min on first run (image pulls).

Monitor progress:
```bash
kubectl --context k3d-ess-helm -n ess get pods -w
```

---

## Step 6: Create first user

```bash
kubectl --context k3d-ess-helm exec -n ess -it \
  deploy/ess-matrix-authentication-service \
  -- mas-cli manage register-user
```

Follow prompts: set username + password.

---

## Step 7: Verify

1. Open `https://element.ess.localhost` in browser
2. Login with user from Step 6
3. Check all pods healthy: `kubectl --context k3d-ess-helm -n ess get pods`

---

## Teardown

```bash
./scripts/destroy_test_cluster.sh
```

---

## Gotchas

| Issue | Fix |
|---|---|
| Browser TLS error | Trust CA (Step 4) or re-run if CA changed |
| Pods stuck Pending | Docker not enough RAM — allocate ≥4 GB in Docker Desktop settings |
| `helm: command not found` | Run Step 1 first |
| Images pull slow | First run only — k3d caches after |
| Context not found | `kubectl config get-contexts` — use `k3d-ess-helm` |

---

## Next: Server deploy

Once Mac local works → fill `decisions.md` → follow `plan.md` Phase 2+.
Key differences for server:
- Real domain + DNS records
- Let's Encrypt certs (not self-signed)
- External PostgreSQL
- `oci://ghcr.io/element-hq/ess-helm/matrix-stack` (not local chart)
