# Deploying ESS Community on an on-prem / self-hosted Ubuntu server

This is a full walkthrough for standing up Element Server Suite Community (the Matrix homeserver stack: Synapse, Matrix Authentication Service, Element Web/Admin, RTC backend, HAProxy, Postgres) on a single Ubuntu box you control — bare metal, home server, or a plain VPS/EC2 instance where you're not using any managed Kubernetes service. Everything runs through K3s on the one machine.

Written from an actual deploy, including the parts that didn't work on the first try.

## What you're actually building

K3s gives you a one-node Kubernetes cluster. Traefik (bundled with K3s) sits on ports 80/443 and routes by hostname to HAProxy, which in turn routes Matrix API traffic to Synapse or MAS depending on the path. Everything else — Postgres, Redis, the RTC/SFU media server, Element's web clients — runs as pods alongside them. cert-manager talks to Let's Encrypt and keeps the TLS certs current without you touching them again.

None of this needs cloud APIs. If the box has a public IP and you can open a few ports, this works the same on a home server as it does on EC2.

## Before you start

**Hardware.** 2 cores / 2GB RAM is the floor, but you'll be swapping constantly. 4 cores and 8GB is a saner minimum once Synapse, Postgres and the RTC service are all running together. Give it real disk too — 20GB is not enough headroom once you account for container images plus a growing Postgres + media volume; go for 40-50GB+ if you can. Default cloud images often ship with a tiny root volume (8GB is common on some AWS AMIs) — check `df -h /` before you install anything, it's much easier to grow the disk now than after K3s has already written a gigabyte of image layers to it.

If you do need to grow an EBS-backed root volume after the fact: resize the volume in the AWS console (or `aws ec2 modify-volume`), then on the instance:

```bash
lsblk                              # confirm the disk shows the new, bigger size
sudo growpart /dev/nvme0n1 1       # or /dev/xvda 1 on older instance types — check lsblk output
sudo resize2fs /dev/nvme0n1p1      # match the partition name from growpart's output
df -h /                            # should show the new size now
```

This works live, no reboot, no downtime.

**A domain, or not.** You need something to be the "server name" — the part after the `:` in a Matrix ID (`@alice:example.com`). If you own a domain, use it and point DNS records at your public IP:

```
example.com          A  <your public IP>
synapse.example.com  A  <your public IP>
account.example.com  A  <your public IP>
mrtc.example.com      A  <your public IP>
element.example.com  A  <your public IP>
admin.example.com    A  <your public IP>
```

If you're just testing and don't want to buy a domain yet, `nip.io` resolves any `<anything>.<your-ip-with-dashes>.nip.io` straight to that IP with zero DNS setup — e.g. if your IP is `98.80.225.142`, then `element.98-80-225-142.nip.io` just works. Fine for a test box, not something you'd want long-term (the server name can't be changed later without wiping the database, so treat this as throwaway if you go this route).

**Ports.** Open on the host and in whatever's in front of it (router, cloud security group):

| Port | Proto | For |
|---|---|---|
| 22 | tcp | SSH |
| 80 | tcp | HTTP → HTTPS redirect, and Let's Encrypt's HTTP-01 challenge |
| 443 | tcp | everything else |
| 30001 | tcp | RTC/SFU, WebRTC TCP fallback |
| 30002 | udp | RTC/SFU, WebRTC media |

If you're on a cloud provider, this is a security group setting, not a `ufw` setting — `ufw` only matters for traffic that already reached the box.

## 1. Base OS

```bash
sudo apt-get update && sudo apt-get upgrade -y
sudo apt-get install -y curl git jq ufw
```

Firewall, if you're using one on the host itself:

```bash
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw allow 30001/tcp
sudo ufw allow 30002/udp
sudo ufw enable
```

## 2. K3s

```bash
curl -sfL https://get.k3s.io | sh -
```

Give yourself kubectl access instead of always sudo'ing:

```bash
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown "$USER:$USER" ~/.kube/config
chmod 600 ~/.kube/config
echo 'export KUBECONFIG=~/.kube/config' >> ~/.bashrc
export KUBECONFIG=~/.kube/config

kubectl get nodes
```

Should show one node, `Ready`.

## 3. Helm

```bash
curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version --short
```

## 4. cert-manager + Let's Encrypt

```bash
helm upgrade --install cert-manager \
  oci://quay.io/jetstack/charts/cert-manager \
  --namespace cert-manager --create-namespace \
  --version v1.17.0 \
  --set crds.enabled=true \
  --timeout 10m --wait

kubectl get pods -n cert-manager   # 3 pods, all 1/1 Running
```

Create both a staging and a production issuer. Deploy against staging first — it proves the HTTP-01 challenge path actually reaches your box (i.e. your ports/DNS/security-group setup is right) without burning a real Let's Encrypt request or risking their rate limit while you're still debugging:

```bash
kubectl apply -f - <<EOF
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    email: you@example.com
    privateKeySecretRef:
      name: letsencrypt-staging
    solvers:
    - http01:
        ingress:
          class: traefik
EOF

kubectl apply -f - <<EOF
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: you@example.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: traefik
EOF
```

## 5. Values files

```bash
mkdir -p ~/ess-values
kubectl create namespace ess
```

`~/ess-values/hostnames.yaml` — mind the schema here, it's nested under `ingress:`, not a flat `hostname:` key (older docs floating around show the flat form, it'll get rejected by the chart's schema validation with an "additional properties not allowed" error):

```yaml
serverName: example.com

synapse:
  ingress:
    host: synapse.example.com

matrixAuthenticationService:
  ingress:
    host: account.example.com

matrixRTC:
  ingress:
    host: mrtc.example.com

elementWeb:
  ingress:
    host: element.example.com

elementAdmin:
  ingress:
    host: admin.example.com
```

If the exact shape ever drifts again, the source of truth is `charts/matrix-stack/ci/fragments/quick-setup-hostnames.yaml` in this repo, not any prose doc — copy from there directly if in doubt.

`~/ess-values/tls.yaml` — start on staging:

```yaml
certManager:
  clusterIssuer: letsencrypt-staging
```

By default the chart deploys its own Postgres. That's fine to get running and to test with; for anything you actually care about, point it at a Postgres instance you manage and back up yourself (see `docs/advanced.md`).

## 6. Deploy

```bash
helm upgrade --install ess oci://ghcr.io/element-hq/ess-helm/matrix-stack \
  --namespace ess \
  -f ~/ess-values/hostnames.yaml \
  -f ~/ess-values/tls.yaml \
  --wait --timeout 20m

kubectl get pods -n ess
```

Once everything's up and the staging certs issued fine (`kubectl get certificate -n ess` — all `READY True`), switch to prod and re-run the same command with `clusterIssuer: letsencrypt-prod` in `tls.yaml`. Helm will just re-issue the real certs over the staging ones.

## 7. First users

Registration is closed by default on purpose — an internet-facing Matrix server with open registration gets found and farmed for spam accounts within hours.

```bash
kubectl exec -n ess -it deploy/ess-matrix-authentication-service -- mas-cli manage register-user
```

Or non-interactively:

```bash
kubectl exec -n ess deploy/ess-matrix-authentication-service -- \
  mas-cli manage register-user --yes --password 'something-strong' alice
```

To let people register themselves later, MAS needs SMTP configured (see `docs/advanced.md`) — don't turn off email verification without also turning on registration tokens, or you're back to the spam problem.

## 8. Actually check it works

```bash
curl https://example.com/.well-known/matrix/client
curl https://example.com/.well-known/matrix/server
```

Both should return JSON pointing at your `synapse.` and `mrtc.` hosts.

Federation check: `https://federationtester.matrix.org/#example.com` (or hit its API directly: `https://federationtester.matrix.org/api/report?server_name=example.com` and check `FederationOK` is `true`).

MAS is reachable and speaking OIDC if `https://account.example.com/.well-known/openid-configuration` returns something.

The actual login has to happen in a browser — MAS is authorization_code-flow-only, there's no password endpoint you can curl to prove login end-to-end. Open `https://element.example.com` and log in with the account you made.

## Upgrading later

Same command as the install, run again:

```bash
helm upgrade ess oci://ghcr.io/element-hq/ess-helm/matrix-stack \
  --namespace ess \
  -f ~/ess-values/hostnames.yaml \
  -f ~/ess-values/tls.yaml \
  --wait --timeout 20m
```

Check `CHANGELOG.md` before jumping major versions.

## Backups

There's no cloud snapshot layer doing this for you here — it's on you:

- Postgres: `kubectl exec -n ess deploy/ess-postgresql -- pg_dumpall -U postgres > backup.sql`, cron it, copy it off the box.
- The generated secrets: `kubectl get secret ess-generated -n ess -o yaml > ess-generated-secret.yaml`. Needed to recover without regenerating every credential from scratch.
- Media uploads live on the `ess-synapse-media` PVC, backed by local disk. Snapshot at the filesystem level or rsync it somewhere else. If a single disk failure would also wipe your only backup, that's not a backup.

## Production checklist, if this stops being a test box

- [ ] External Postgres, not the bundled one, with its own backup schedule
- [ ] Media storage off local disk (S3-compatible or NFS) if you need it to survive a disk failure
- [ ] SMTP on MAS so real registration/password-reset works
- [ ] Firewall/security-group rules trimmed to just what's needed — don't leave a wide port range open for the sake of it
- [ ] Prometheus Operator installed, if you want the `ServiceMonitor`s the chart already creates to do anything
- [ ] Automated, off-host backups actually running, not just documented
- [ ] Some thought given to what happens if the box loses power mid-write — single node, no failover

## Troubleshooting

```bash
kubectl get pods -n ess
kubectl logs -n ess deploy/ess-synapse
kubectl logs -n ess deploy/ess-matrix-authentication-service
kubectl logs -n ess deploy/ess-matrix-rtc-sfu
kubectl describe certificate -n ess     # cert-manager / Let's Encrypt stuck? start here
```

More scenarios in `docs/troubleshooting.md`.

## Tearing it down

```bash
helm uninstall ess -n ess
kubectl delete secrets/ess-generated -n ess
kubectl delete configmap/ess-deployment-markers -n ess
kubectl delete pvc/ess-synapse-media -n ess
kubectl delete pvc/ess-postgres-data -n ess
# or just wipe the whole namespace:
kubectl delete namespace ess

helm uninstall cert-manager -n cert-manager
rm -rf /usr/local/bin/helm $HOME/.cache/helm $HOME/.config/helm $HOME/.local/share/helm
/usr/local/bin/k3s-uninstall.sh
rm -rf ~/ess-values ~/.kube
```
