# ESS Community — AWS EC2 Deploy (Ubuntu)

Deploy the full Element Server Suite on an AWS EC2 instance running Ubuntu. Real domain, Let's Encrypt TLS, single-node K3s.

---

## Architecture

```
Internet → Route 53 / DNS → EC2 Public IP
                              ↓
                         K3s + Traefik
                              ↓
                     ESS (Synapse, MAS, Element Web, RTC, ...)
```

---

## Prerequisites

### 1. EC2 instance

| Setting | Value |
|---|---|
| AMI | Ubuntu 24.04 LTS (x86_64) |
| Instance type | `t3.medium` (2 vCPU, 4 GB RAM) minimum; `t3.large` recommended |
| Storage | 30 GB gp3 root volume |
| Security group | See below |

Security group inbound rules:

| Port | Protocol | Source | Purpose |
|---|---|---|---|
| 22 | TCP | Your IP | SSH |
| 80 | TCP | 0.0.0.0/0 | HTTP (Let's Encrypt challenge) |
| 443 | TCP | 0.0.0.0/0 | HTTPS |
| 30000-30100 | UDP | 0.0.0.0/0 | Matrix RTC (voice/video) |
| 30000-30100 | TCP | 0.0.0.0/0 | Matrix RTC (voice/video TCP fallback) |

### 2. Domain

You need a domain where you control DNS. Example: `matrix.example.com`

Add these DNS A records pointing to your EC2 public IP:

```
example.com          A  <EC2_PUBLIC_IP>   # server name + .well-known
synapse.example.com  A  <EC2_PUBLIC_IP>   # Synapse
mas.example.com      A  <EC2_PUBLIC_IP>   # Matrix Authentication Service
mrtc.example.com     A  <EC2_PUBLIC_IP>   # Matrix RTC
element.example.com  A  <EC2_PUBLIC_IP>   # Element Web
admin.example.com    A  <EC2_PUBLIC_IP>   # Element Admin
```

> Wait for DNS propagation before proceeding (check with `dig element.example.com`).

---

## Step 1 — SSH into EC2

```bash
ssh -i your-key.pem ubuntu@<EC2_PUBLIC_IP>
```

---

## Step 2 — System prep

```bash
sudo apt-get update && sudo apt-get upgrade -y
sudo apt-get install -y curl git jq
```

---

## Step 3 — Install K3s

```bash
curl -sfL https://get.k3s.io | sh -

# Give ubuntu user access
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown ubuntu:ubuntu ~/.kube/config
chmod 600 ~/.kube/config

# Verify
kubectl get nodes
# NAME        STATUS   ROLES                  AGE   VERSION
# <hostname>  Ready    control-plane,master   30s   v1.3x.x
```

---

## Step 4 — Install Helm 3

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version --short   # must show v3.x.x
```

---

## Step 5 — Clone this repo

```bash
git clone https://github.com/albaghel/ess-helm.git
cd ess-helm
```

---

## Step 6 — Install cert-manager

```bash
helm upgrade --install cert-manager \
  oci://quay.io/jetstack/charts/cert-manager \
  --namespace cert-manager --create-namespace \
  --version v1.20.3 \
  --set crds.enabled=true \
  --timeout 10m --wait

kubectl get pods -n cert-manager
# All 3 pods must be 1/1 Running
```

---

## Step 7 — Let's Encrypt ClusterIssuer

Replace `your@email.com` with a real email (LE sends cert expiry notices):

```bash
kubectl apply -f - <<EOF
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: your@email.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: traefik
EOF
```

Test with staging first (avoids LE rate limits):
```bash
# Replace acme server with: https://acme-staging-v02.api.letsencrypt.org/directory
# Use name: letsencrypt-staging everywhere instead
```

---

## Step 8 — ESS values files

Create `~/ess-values/` to store your config:

```bash
mkdir -p ~/ess-values
```

### hostnames.yaml

```bash
cat > ~/ess-values/hostnames.yaml <<EOF
serverName: example.com

synapse:
  hostname: synapse.example.com

matrixAuthenticationService:
  hostname: mas.example.com

matrixRTC:
  hostname: mrtc.example.com

elementWeb:
  hostname: element.example.com

elementAdmin:
  hostname: admin.example.com
EOF
```

### tls.yaml

```bash
cat > ~/ess-values/tls.yaml <<EOF
certManager:
  clusterIssuer: letsencrypt-prod
EOF
```

---

## Step 9 — Deploy ESS

```bash
kubectl create namespace ess

helm upgrade --install ess charts/matrix-stack \
  --namespace ess \
  -f ~/ess-values/hostnames.yaml \
  -f ~/ess-values/tls.yaml \
  --wait --timeout 20m
```

Watch pods come up:
```bash
kubectl get pods -n ess -w
```

---

## Step 10 — Create first user

```bash
kubectl exec -n ess -it deployment/ess-matrix-authentication-service \
  -- mas-cli manage register-user
```

Or non-interactively:
```bash
kubectl exec -n ess deployment/ess-matrix-authentication-service \
  -- mas-cli manage register-user --yes --password "YourPassword123!" yourusername
```

---

## Step 11 — Verify

Open `https://element.example.com` in a browser. TLS should show valid (Let's Encrypt).

Test federation:
```
https://federationtester.matrix.org/#example.com
```

---

## Upgrades

```bash
cd ess-helm
git pull upstream main      # get latest chart
helm upgrade ess charts/matrix-stack \
  --namespace ess \
  -f ~/ess-values/hostnames.yaml \
  -f ~/ess-values/tls.yaml \
  --wait --timeout 20m
```

---

## Backups

Critical data to back up:

| What | Where |
|---|---|
| PostgreSQL | `kubectl exec -n ess deploy/ess-postgresql -- pg_dumpall -U postgres` |
| Generated secrets | `kubectl get secret ess-generated -n ess -o yaml` |
| Media files | PVC `ess-synapse-media` — snapshot EBS or use S3 (see `docs/advanced.md`) |

---

## Production hardening (post-smoke-test)

- [ ] External PostgreSQL (RDS) — remove bundled DB, enables snapshots
- [ ] S3 media storage — `synapse.media.s3` in values (see `docs/advanced.md`)
- [ ] SMTP config — enables user self-registration via MAS
- [ ] Restrict security group — tighten NodePort range to MRTC ports only
- [ ] Monitoring — ServiceMonitor + Prometheus Operator
- [ ] Automatic DB backups — cron `pg_dump` to S3

---

## Troubleshooting

```bash
# All pods status
kubectl get pods -n ess

# Specific pod logs
kubectl logs -n ess deployment/ess-synapse
kubectl logs -n ess deployment/ess-matrix-authentication-service
kubectl logs -n ess deployment/ess-matrix-rtc-sfu

# cert-manager certificate status
kubectl describe certificate -n ess

# Check .well-known
curl https://example.com/.well-known/matrix/client
curl https://example.com/.well-known/matrix/server
```

See `docs/troubleshooting.md` for more.
