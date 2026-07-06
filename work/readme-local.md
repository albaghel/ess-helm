# ESS Community — Local Mac Deploy Guide

Deploy Element Server Suite Community on Mac using k3d (K3s in Docker).
No real domain. No public DNS. Self-signed TLS. Full Matrix stack on `*.ess.localhost`.

---

## Prerequisites

| Tool | Install | Notes |
|---|---|---|
| Docker Desktop | Already installed | Must be running |
| kubectl | Already installed | |
| uv | Already installed | |
| helm | `brew install helm@3 && brew link --force helm@3` | Must be Helm 3, NOT 4 |
| k3d | `brew install k3d` | |

Verify:
```bash
helm version --short   # must show v3.x.x
k3d version
docker ps
```

---

## Step 1 — Create k3d cluster

Use `work/k3d-simple.yaml` (NOT `scripts/setup_test_cluster.sh` — that script's audit policy breaks cert-manager on Mac).

```bash
DOCKER_HOST=unix:///Users/Alkes/.docker/run/docker.sock \
  k3d cluster create --config work/k3d-simple.yaml

DOCKER_HOST=unix:///Users/Alkes/.docker/run/docker.sock \
  k3d kubeconfig merge ess-helm -d
```

Verify:
```bash
kubectl --context k3d-ess-helm get nodes
# k3d-ess-helm-server-0   Ready   control-plane
```

---

## Step 2 — Install cert-manager

Must install manually (pyhelm3 in setup script is incompatible with OCI pull output).

```bash
helm --kube-context k3d-ess-helm upgrade --install cert-manager \
  oci://quay.io/jetstack/charts/cert-manager \
  --namespace cert-manager --create-namespace \
  --version v1.20.3 \
  --set crds.enabled=true \
  --timeout 10m --wait
```

Wait for all 3 pods ready (takes ~30s):
```bash
kubectl --context k3d-ess-helm get pods -n cert-manager
# cert-manager-xxx             1/1  Running
# cert-manager-cainjector-xxx  1/1  Running
# cert-manager-webhook-xxx     1/1  Running  ← must be 1/1
```

---

## Step 3 — Set up self-signed CA

```bash
# Self-signed root ClusterIssuer
kubectl --context k3d-ess-helm apply -f - <<'EOF'
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: ess-ca
spec:
  selfSigned: {}
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: ess-ca
  namespace: cert-manager
spec:
  isCA: true
  commonName: ess-ca
  secretName: ess-ca
  duration: 87660h0m0s
  privateKey:
    algorithm: RSA
  issuerRef:
    name: ess-ca
    kind: ClusterIssuer
    group: cert-manager.io
EOF

# Wait for CA cert
until kubectl --context k3d-ess-helm get certificate ess-ca -n cert-manager \
  -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}' | grep -q True; do sleep 2; done

# ClusterIssuer backed by CA
kubectl --context k3d-ess-helm apply -f - <<'EOF'
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: ess-selfsigned
spec:
  ca:
    secretName: ess-ca
EOF
```

---

## Step 4 — Create ESS namespace + wildcard cert

```bash
kubectl --context k3d-ess-helm create namespace ess

kubectl --context k3d-ess-helm apply -f - <<'EOF'
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: ess-wildcard
  namespace: ess
spec:
  secretName: ess-wildcard-tls
  issuerRef:
    name: ess-selfsigned
    kind: ClusterIssuer
  commonName: ess.localhost
  dnsNames:
  - ess.localhost
  - "*.ess.localhost"
EOF

until kubectl --context k3d-ess-helm get certificate ess-wildcard -n ess \
  -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}' | grep -q True; do sleep 2; done
```

---

## Step 5 — Save CA cert

```bash
mkdir -p ~/.config/ess-helm-ca
kubectl --context k3d-ess-helm get secret ess-ca -n cert-manager \
  -o jsonpath='{.data.ca\.crt}' | base64 -d > ~/.config/ess-helm-ca/ca.crt
```

---

## Step 6 — Deploy ESS

```bash
helm -n ess --kube-context k3d-ess-helm upgrade -i ess charts/matrix-stack \
  -f charts/matrix-stack/ci/test-cluster-mixin.yaml \
  -f charts/matrix-stack/ci/example-default-enabled-components-values.yaml \
  --wait --timeout 15m
```

Takes 2–5 min (image pulls on first run). All pods should reach Running:
```bash
kubectl --context k3d-ess-helm get pods -n ess
```

---

## Step 7 — Create first user

```bash
kubectl --context k3d-ess-helm exec -n ess \
  deployment/ess-matrix-authentication-service \
  -- mas-cli manage register-user --yes --password "YourPassword123!" yourusername
```

Password must be strong (uppercase + number + symbol).

---

## Step 8 — Trust CA in browser

Run in **Terminal.app** (not here — needs interactive sudo):
```bash
sudo security add-trusted-cert -d -r trustRoot \
  -k /Library/Keychains/System.keychain \
  /Users/Alkes/.config/ess-helm-ca/ca.crt
```

Then **fully quit** (Cmd+Q) and reopen your browser.

> Firefox: `about:preferences#privacy` → Certificates → View Certificates → Authorities → Import → select `/Users/Alkes/.config/ess-helm-ca/ca.crt` → Trust for websites.

---

## Step 9 — Open Element Web

| URL | Service |
|---|---|
| `https://element.ess.localhost` | Element Web (chat) |
| `https://admin.ess.localhost` | Element Admin |
| `https://mas.ess.localhost` | Matrix Authentication Service |
| `https://synapse.ess.localhost` | Synapse API |

Login at `https://element.ess.localhost` with the user created in Step 7.

---

## Teardown

```bash
DOCKER_HOST=unix:///Users/Alkes/.docker/run/docker.sock \
  ./scripts/destroy_test_cluster.sh
```

Recreating: repeat from Step 1. CA secret in `~/.config/ess-helm-ca/` persists — Step 3 will reuse it.

---

## Known gotchas

| Issue | Cause | Fix |
|---|---|---|
| `setup_test_cluster.sh` fails | Audit policy `level: Metadata` floods API server → cert-manager informer timeout | Use `work/k3d-simple.yaml` instead |
| cert-manager webhook `0/1` forever | Audit logging overloads API server | Clean cluster without audit args |
| `helm: command not found` | Helm 4 installed via brew | `brew install helm@3 && brew link --force helm@3` |
| k3d can't reach Docker | Docker Desktop uses non-default socket | Prefix commands with `DOCKER_HOST=unix:///Users/Alkes/.docker/run/docker.sock` |
| `sudo security` error: `/var/root/...` | `~` expands to `/var/root` under sudo | Use full path `/Users/Alkes/.config/ess-helm-ca/ca.crt` |
| Element Web "misconfigured" | Browser not trusting CA → AJAX to Synapse blocked | Complete Step 8 (CA trust) |
| "Confirm digital identity" on first login | Normal MAS/E2E prompt | Generate security key or skip for dev |
