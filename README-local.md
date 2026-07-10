# ESS Community — Local Mac Deploy (k3d)

Deploy the full Element Server Suite (Matrix homeserver stack) locally on macOS using k3d (K3s in Docker). No real domain, no public DNS, self-signed TLS, everything runs on `*.ess.localhost`.

---

## Stack

| Component | Role |
|---|---|
| Synapse | Matrix homeserver |
| Matrix Authentication Service (MAS) | OIDC-based auth |
| HAProxy | Routing + `.well-known` |
| PostgreSQL | Database (bundled) |
| Element Web | Web chat client |
| Element Admin | Admin console |
| Matrix RTC (LiveKit SFU) | Voice/video calls |
| Redis | Synapse worker pub/sub |

---

## Prerequisites

| Tool | Install | Notes |
|---|---|---|
| Docker Desktop | [docker.com](https://docker.com) | Must be running |
| kubectl | `brew install kubectl` | Any recent |
| Helm 3 | `brew install helm@3 && brew link --force helm@3` | **Must be 3.x, NOT 4** |
| k3d | `brew install k3d` | v5+ |

```bash
helm version --short   # must show v3.x.x
k3d version
docker ps
```

---

## Step 1 — Create k3d cluster

> Do NOT use `scripts/setup_test_cluster.sh` — its audit policy (`level: Metadata`) floods the API server and breaks cert-manager on Mac.

```bash
DOCKER_HOST=unix:///Users/$USER/.docker/run/docker.sock \
  k3d cluster create --config work/k3d-simple.yaml

DOCKER_HOST=unix:///Users/$USER/.docker/run/docker.sock \
  k3d kubeconfig merge ess-helm -d
```

Verify:
```bash
kubectl --context k3d-ess-helm get nodes
# k3d-ess-helm-server-0   Ready   control-plane
```

---

## Step 2 — Install cert-manager

Must install manually (setup script's pyhelm3 incompatible with OCI pull output format).

```bash
helm --kube-context k3d-ess-helm upgrade --install cert-manager \
  oci://quay.io/jetstack/charts/cert-manager \
  --namespace cert-manager --create-namespace \
  --version v1.20.3 \
  --set crds.enabled=true \
  --timeout 10m --wait
```

Confirm all 3 pods ready:
```bash
kubectl --context k3d-ess-helm get pods -n cert-manager
# cert-manager-xxx             1/1  Running
# cert-manager-cainjector-xxx  1/1  Running
# cert-manager-webhook-xxx     1/1  Running  ← must be 1/1
```

---

## Step 3 — Self-signed CA

```bash
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

until kubectl --context k3d-ess-helm get certificate ess-ca -n cert-manager \
  -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}' | grep -q True; do sleep 2; done

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

## Step 4 — ESS namespace + wildcard cert

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
  -f work/local-rtc-fix.yaml \
  --wait --timeout 15m
```

After deploy, patch SFU configmap so voice/video uses `127.0.0.1` (fixes call drops):

```bash
kubectl --context k3d-ess-helm patch configmap ess-matrix-rtc-sfu -n ess --type=json -p='[
  {
    "op": "replace",
    "path": "/data/config-overrides.yaml",
    "value": "port: 7880\n\nprometheus:\n  port: 6789\n\nlogging:\n  level: info\n  pion_level: error\n  json: false\n\nrtc:\n  use_external_ip: false\n  node_ip: \"127.0.0.1\"\n  tcp_port: 30001\n  udp_port: 30002\n\nkey_file: /conf/keys.yaml\n\nroom:\n  auto_create: false\n"
  }
]'

kubectl --context k3d-ess-helm rollout restart deployment/ess-matrix-rtc-sfu -n ess
kubectl --context k3d-ess-helm rollout status deployment/ess-matrix-rtc-sfu -n ess
```

> **Note:** `helm upgrade` reverts the configmap patch — rerun patch + rollout restart after every upgrade.

Verify all pods:
```bash
kubectl --context k3d-ess-helm get pods -n ess
```

---

## Step 7 — Create users

```bash
kubectl --context k3d-ess-helm exec -n ess \
  deployment/ess-matrix-authentication-service \
  -- mas-cli manage register-user --yes --password "YourPassword123!" yourusername
```

Password must have uppercase, number, and symbol.

---

## Step 8 — Trust CA cert (run in Terminal.app)

```bash
sudo security add-trusted-cert -d -r trustRoot \
  -k /Library/Keychains/System.keychain \
  /Users/$USER/.config/ess-helm-ca/ca.crt
```

Fully quit browser (Cmd+Q) then reopen.

**Firefox alternative** (no sudo): `about:preferences#privacy` → Certificates → View Certificates → Authorities → Import → select `~/.config/ess-helm-ca/ca.crt` → Trust for websites.

---

## Step 9 — Open Element Web

| URL | Service |
|---|---|
| `https://element.ess.localhost` | Element Web (chat + voice/video) |
| `https://admin.ess.localhost` | Element Admin |
| `https://mas.ess.localhost` | Matrix Authentication Service |
| `https://synapse.ess.localhost` | Synapse API |

---

## Teardown

```bash
DOCKER_HOST=unix:///Users/$USER/.docker/run/docker.sock \
  k3d cluster delete ess-helm
```

---

## Known gotchas

| Issue | Cause | Fix |
|---|---|---|
| `setup_test_cluster.sh` breaks cert-manager | Audit policy `level: Metadata` overloads API server → informer timeout | Use `work/k3d-simple.yaml` |
| cert-manager webhook stuck `0/1` | Same audit logging issue | Recreate cluster without audit args |
| Helm `json.decoder.JSONDecodeError` | Helm 4.x OCI output format changed; pyhelm3 incompatible | `brew install helm@3 && brew link --force helm@3` |
| k3d can't find Docker | Docker Desktop uses non-default socket | Prefix: `DOCKER_HOST=unix:///Users/$USER/.docker/run/docker.sock` |
| `sudo security` fails: `/var/root/...` | `~` expands to `/var/root` under sudo | Use full path: `/Users/$USER/.config/ess-helm-ca/ca.crt` |
| Element Web "misconfigured" | Browser rejects self-signed cert → AJAX to Synapse blocked | Complete Step 8 |
| Voice/video drops: `UNKNOWN_ERROR` | SFU advertises public/Docker IP instead of `127.0.0.1` | Apply SFU configmap patch in Step 6 |
| SFU patch reverted | `helm upgrade` regenerates configmap | Reapply patch + restart after every upgrade |
| "Confirm digital identity" on first login | Normal MAS/E2E encryption setup | Generate security key or skip for dev |
