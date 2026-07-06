# ESS Community — On-Premise Ubuntu Server Deploy

Deploy the full Element Server Suite (Matrix homeserver stack) on a **self-hosted, on-premise Ubuntu server** — your own hardware/VM, not a cloud provider. Covers everything start to end: OS prep, networking, K3s (single-node Kubernetes), TLS, the Helm install, and how the whole system fits together.

---

## 1. Architecture — how this project works

ESS Community is a **Helm chart** (`charts/matrix-stack`) that deploys a full Matrix homeserver as a set of pods on a **Kubernetes cluster**. On-prem, that cluster is a single-node **K3s** install running directly on your Ubuntu box — no cloud APIs, no managed services, everything lives on your hardware.

```
                         Your Ubuntu Server (bare metal / VM)
                         ┌─────────────────────────────────────────────┐
Internet/LAN ── 80/443 ──┤  K3s (single-node Kubernetes)                │
   │                     │                                             │
   │                     │  Traefik (Ingress) ──┐                      │
   │                     │                       ▼                     │
   │                     │   ┌────────────── HAProxy ─────────────┐    │
   │                     │   │  routes /_matrix, /.well-known,    │    │
   │                     │   │  auth endpoints to the right pod    │    │
   │                     │   └──────┬──────────┬──────────┬───────┘    │
   │                     │          ▼          ▼          ▼            │
   │                     │      Synapse      MAS      Element Web /    │
   │                     │    (Matrix HS)  (OIDC auth)  Admin (static)  │
   │                     │          │          │                       │
   │                     │          ▼          ▼                       │
   │                     │       PostgreSQL (bundled, or external)     │
   │                     │          ▲                                 │
   │                     │      Redis (Synapse worker pub/sub)         │
   │                     │                                             │
 30001/tcp,30002/udp ────┤  Matrix RTC Backend (LiveKit SFU) — calls   │
                         └─────────────────────────────────────────────┘
```

**Component roles:**

| Component | Role |
|---|---|
| **K3s** | Lightweight Kubernetes distribution. Runs all ESS pods, manages storage, networking, and the built-in Traefik ingress/load balancer. |
| **Traefik** | K3s's bundled ingress controller. Terminates TLS (or passes it through) and routes incoming HTTP(S) to the right internal service based on hostname. |
| **cert-manager** | Kubernetes operator that talks to Let's Encrypt (ACME) and auto-issues/renews TLS certs, stored as Kubernetes Secrets. |
| **HAProxy** | Sits behind Traefik. Load-balances Matrix API traffic across Synapse worker processes and MAS, and serves the `/.well-known/matrix` + `/.well-known/element` discovery files needed for federation and client auto-discovery. |
| **Synapse** | The actual Matrix homeserver — handles rooms, events, federation with other Matrix servers, media. Can scale to multiple worker processes. |
| **Redis** | Pub/sub bus letting Synapse's main process and worker processes talk to each other. |
| **Matrix Authentication Service (MAS)** | OIDC-based auth/identity service. Owns login, registration, sessions — Synapse delegates auth to it. |
| **PostgreSQL** | Primary datastore: accounts, room state, messages, device keys, MAS data. Chart bundles one by default; production should point at an external instance you manage/back up yourself. |
| **Element Web** | Browser chat client, preconfigured to point at your homeserver. |
| **Element Admin** | Web-based admin console for the deployment. |
| **Matrix RTC Backend (LiveKit SFU)** | Media server that powers Element Call (voice/video). Needs its own exposed ports (TCP 30001 / UDP 30002) since it's not plain HTTP. |
| **Hookshot** (optional, off by default) | Bridges GitHub/GitLab/JIRA/webhooks into Matrix rooms. |

**Request flow example** (loading Element Web and sending a message):
1. Browser hits `https://element.<server-name>` → DNS resolves to your server's public IP → K3s's Traefik terminates TLS → serves static Element Web assets.
2. Element Web calls `https://<server-name>/.well-known/matrix/client` to discover the homeserver + MAS URLs.
3. Login goes to MAS (`account.<server-name>`) which issues an OIDC token.
4. Element Web then talks to Synapse (`matrix.<server-name>`) using that token for all `/_matrix` API calls — HAProxy routes these to the Synapse (or worker) pod.
5. Synapse persists the message to PostgreSQL and, if other homeservers are in the room, federates the event out over the internet directly from Synapse.

Everything is deployed/upgraded declaratively via `helm upgrade --install`, driven by one or more small YAML "values" files you author (hostnames, TLS, DB config, etc.) — the chart wires the rest together.

---

## 2. Prerequisites

### Hardware / OS

| Requirement | Minimum | Recommended |
|---|---|---|
| OS | Ubuntu 22.04 LTS or 24.04 LTS (x86_64 or arm64) | 24.04 LTS |
| CPU | 2 cores | 4+ cores |
| RAM | 2 GB | 8 GB+ (Synapse + Postgres + RTC add up) |
| Disk | 20 GB free | 100 GB+ SSD (media uploads + Postgres grow over time) |
| Network | Static LAN IP, root/sudo access | Static public IP or DDNS, port-forwarding control on your router/firewall |

### Domain & DNS

You need a domain you control DNS for (a subdomain of one you own is fine). Choose a **server name**, e.g. `example.com` — this becomes the tail of every Matrix ID: `@alice:example.com`.

> **The server name cannot be changed later without recreating the database.** Pick it carefully.

Create these DNS records, all pointing at your server's **public IP** (or the IP your router forwards from, if home-hosted):

```
example.com          A  <YOUR_PUBLIC_IP>   # server name + .well-known
synapse.example.com  A  <YOUR_PUBLIC_IP>
account.example.com  A  <YOUR_PUBLIC_IP>   # Matrix Authentication Service
mrtc.example.com      A  <YOUR_PUBLIC_IP>   # Matrix RTC
element.example.com  A  <YOUR_PUBLIC_IP>   # Element Web
admin.example.com    A  <YOUR_PUBLIC_IP>   # Element Admin
```

Wait for propagation before continuing: `dig element.example.com`.

### Network / firewall ports

Open (forward, if behind a home router/NAT) these on the server:

| Port | Protocol | Purpose |
|---|---|---|
| 22 | TCP | SSH (restrict source IP if possible) |
| 80 | TCP | HTTP → redirects to HTTPS, also used for Let's Encrypt HTTP-01 challenge |
| 443 | TCP | HTTPS for all web-facing services |
| 30001 | TCP | Matrix RTC WebRTC (TCP fallback) |
| 30002 | UDP | Matrix RTC WebRTC (media) |

If the server is behind a NAT router (common on-prem/home scenario), forward these ports from the router to the server's LAN IP, and also open them in `ufw` on the host itself (see below).

---

## 3. Step 1 — Base OS setup

```bash
sudo apt-get update && sudo apt-get upgrade -y
sudo apt-get install -y curl git jq ufw
```

Set a static LAN IP (via netplan or your router's DHCP reservation) so the server's address never changes under K3s.

Configure the firewall:

```bash
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw allow 30001/tcp
sudo ufw allow 30002/udp
sudo ufw enable
sudo ufw status
```

---

## 4. Step 2 — Install K3s (single-node Kubernetes)

```bash
curl -sfL https://get.k3s.io | sh -
```

Give your user kubectl access:

```bash
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown "$USER:$USER" ~/.kube/config
chmod 600 ~/.kube/config
echo 'export KUBECONFIG=~/.kube/config' >> ~/.bashrc
export KUBECONFIG=~/.kube/config

kubectl get nodes
# NAME       STATUS   ROLES                  AGE   VERSION
# <host>     Ready    control-plane,master   30s   v1.3x.x
```

K3s stores persistent volumes under `/var/lib/rancher/k3s/storage/` by default. If your fast disk is mounted elsewhere (e.g. a dedicated SSD/NVMe), either mount it at `/var` before install, or reinstall with:

```bash
export K3S_DATA_DIR=/mnt/fast-disk/k3s
curl -sfL https://get.k3s.io | sh -
```

### (Optional) If port 80/443 is already taken by an existing reverse proxy

If your on-prem box already runs Apache/Nginx/Caddy for other sites, have K3s's Traefik listen on alternate ports instead and forward to it from your existing proxy — see the [main README's "Using an existing reverse proxy" section](README.md#using-an-existing-reverse-proxy) for the full config and example vhosts (Apache2/Nginx/Caddy).

---

## 5. Step 3 — Install Helm 3

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version --short   # must print v3.x.x
```

---

## 6. Step 4 — Install cert-manager + Let's Encrypt

```bash
helm upgrade --install cert-manager \
  oci://quay.io/jetstack/charts/cert-manager \
  --namespace cert-manager --create-namespace \
  --version v1.17.0 \
  --set crds.enabled=true \
  --timeout 10m --wait

kubectl get pods -n cert-manager
# all 3 pods must show 1/1 Running
```

Create the ACME ClusterIssuer (replace the email — Let's Encrypt sends expiry notices there):

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

> To avoid Let's Encrypt rate limits while testing, first point `server` at `https://acme-staging-v02.api.letsencrypt.org/directory` with a `letsencrypt-staging` name, verify it works, then switch to prod.

If you'd rather use your own certificate files instead of Let's Encrypt, see [README.md → Certificate File](README.md#certificate-file).

---

## 7. Step 5 — Configure and deploy ESS

```bash
git clone https://github.com/element-hq/ess-helm.git
cd ess-helm
mkdir -p ~/ess-values
kubectl create namespace ess
```

**`~/ess-values/hostnames.yaml`**:

```bash
cat > ~/ess-values/hostnames.yaml <<EOF
serverName: example.com

synapse:
  hostname: synapse.example.com

matrixAuthenticationService:
  hostname: account.example.com

matrixRTC:
  hostname: mrtc.example.com

elementWeb:
  hostname: element.example.com

elementAdmin:
  hostname: admin.example.com
EOF
```

**`~/ess-values/tls.yaml`**:

```bash
cat > ~/ess-values/tls.yaml <<EOF
certManager:
  clusterIssuer: letsencrypt-prod
EOF
```

By default the chart deploys its own bundled PostgreSQL — fine to start with. For production, point at an external PostgreSQL instance you control (so you own backups/HA) — see [docs/advanced.md → Using a dedicated PostgreSQL database](docs/advanced.md#using-a-dedicated-postgresql-database) and copy `charts/matrix-stack/ci/fragments/quick-setup-postgresql.yaml` to `~/ess-values/postgresql.yaml`.

Deploy:

```bash
helm upgrade --install ess oci://ghcr.io/element-hq/ess-helm/matrix-stack \
  --namespace ess \
  -f ~/ess-values/hostnames.yaml \
  -f ~/ess-values/tls.yaml \
  --wait --timeout 20m

kubectl get pods -n ess -w
```

(Using the chart straight from the OCI registry, as above, is simpler for upgrades than deploying from the cloned repo path — pick one and stay consistent.)

---

## 8. Step 6 — Create your first user

Registration is closed by default (prevents spam bots from finding your fresh server).

```bash
kubectl exec -n ess -it deploy/ess-matrix-authentication-service -- mas-cli manage register-user
```

Non-interactive:

```bash
kubectl exec -n ess deploy/ess-matrix-authentication-service \
  -- mas-cli manage register-user --yes --password "YourStrongPassw0rd!" alice
```

To let users self-register later, configure MAS with SMTP — see [docs/advanced.md → Configuring Matrix Authentication Service](docs/advanced.md#configuring-matrix-authentication-service). Never set `password_registration_email_required: false` without `registration_token_required: true`, or your server will get farmed for spam accounts.

---

## 9. Step 7 — Verify

```bash
curl https://example.com/.well-known/matrix/client
curl https://example.com/.well-known/matrix/server
```

- Open `https://element.example.com` and log in with the user you created.
- Check federation: `https://federationtester.matrix.org/#example.com`
- Log in from an Element X mobile client.
- (Optional) install [k9s](https://k9scli.io/) for a terminal UI over the cluster: `sudo snap install k9s`.

---

## 10. Upgrades

```bash
helm upgrade ess oci://ghcr.io/element-hq/ess-helm/matrix-stack \
  --namespace ess \
  -f ~/ess-values/hostnames.yaml \
  -f ~/ess-values/tls.yaml \
  --wait --timeout 20m
```

Check the chart's release notes / `CHANGELOG.md` before upgrading across major versions.

---

## 11. Backups (your responsibility on-prem — no cloud snapshots)

| What | How |
|---|---|
| PostgreSQL | `kubectl exec -n ess deploy/ess-postgresql -- pg_dumpall -U postgres > backup.sql`, cron this to local disk + copy off-box (NAS, another machine, offsite). |
| Generated secrets | `kubectl get secret ess-generated -n ess -o yaml > ess-generated-secret.yaml` — store securely, needed to recover without regenerating creds. |
| Media uploads | Persistent volume `ess-synapse-media`, backed by local disk under K3s's storage path. Snapshot at the filesystem/LVM level, or rsync to a NAS, or configure S3-compatible media storage instead (see [docs/advanced.md](docs/advanced.md)). |

Since there's no cloud storage layer here, put backups on a **second physical disk or another machine** — a single-disk failure otherwise takes out both the live data and the backup.

---

## 12. Production hardening checklist

- [ ] External PostgreSQL instance (not the bundled one) with its own backup schedule
- [ ] S3-compatible (e.g. MinIO on a separate box) or NFS media storage instead of local PVC, if you need HA or off-host storage
- [ ] SMTP configured on MAS, so real self-registration works
- [ ] `ufw`/router rules restricted to only the ports actually needed (30001-30002 only for RTC — don't open a wide range)
- [ ] Prometheus Operator installed → chart auto-creates `ServiceMonitor`s for metrics
- [ ] Automated `pg_dump` + media backup cron, copied off-host
- [ ] UPS / power protection — a single-node cluster has no failover if the box loses power mid-write

---

## 13. Troubleshooting

```bash
kubectl get pods -n ess
kubectl logs -n ess deploy/ess-synapse
kubectl logs -n ess deploy/ess-matrix-authentication-service
kubectl logs -n ess deploy/ess-matrix-rtc-sfu
kubectl describe certificate -n ess     # cert-manager / Let's Encrypt issues
```

More scenarios: [docs/troubleshooting.md](docs/troubleshooting.md).

---

## 14. Uninstalling

```bash
helm uninstall ess -n ess
kubectl delete secrets/ess-generated -n ess
kubectl delete configmap/ess-deployment-markers -n ess
kubectl delete pvc/ess-synapse-media -n ess
kubectl delete pvc/ess-postgres-data -n ess
# or, to remove everything at once:
kubectl delete namespace ess

# Remove cert-manager
helm uninstall cert-manager -n cert-manager

# Remove Helm
rm -rf /usr/local/bin/helm $HOME/.cache/helm $HOME/.config/helm $HOME/.local/share/helm

# Remove K3s
/usr/local/bin/k3s-uninstall.sh

# Remove local config
rm -rf ~/ess-values ~/.kube
```
</content>
