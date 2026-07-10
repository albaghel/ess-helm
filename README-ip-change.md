# Changing the public IP (new server, or IP just changed)

Covers two cases: moving ESS to a brand-new box, or the same box getting a new public IP (re-provisioned VPS, EC2 instance stopped/started without an Elastic IP, ISP changed your home IP, etc). Either way the fix is the same — DNS has to catch up to the new IP, then TLS re-validates itself.

Assumes you already have a working deploy per `README-onprem.md`. `serverName` (the Matrix server name, e.g. `example.com`) is **not** changing here — only the IP it resolves to. If `serverName` itself changes, that's a much bigger job (new server identity, no clean migration path, effectively a fresh server for federation purposes) and out of scope for this doc.

## 1. Update DNS first

Point every A record at the new IP:

```
example.com          A  <new public IP>
synapse.example.com  A  <new public IP>
account.example.com  A  <new public IP>
mrtc.example.com      A  <new public IP>
element.example.com  A  <new public IP>
admin.example.com    A  <new public IP>
```

Do this before anything else — the rest of this doc depends on DNS already pointing at the new box. Check it's actually propagated before moving on:

```bash
dig +short example.com
dig +short synapse.example.com
```

Both should return the new IP. If you're on a low TTL this is fast; if you inherited a high TTL from the old setup, wait it out here rather than fighting it.

## 2. Open ports on the new box / new security group

Same set as the original deploy — nothing new:

| Port | Proto | For |
|---|---|---|
| 22 | tcp | SSH |
| 80 | tcp | HTTP → HTTPS redirect, Let's Encrypt HTTP-01 challenge |
| 443 | tcp | everything else |
| 30001 | tcp | RTC/SFU, WebRTC TCP fallback |
| 30002 | udp | RTC/SFU, WebRTC media |

Cloud provider: security group setting. Home server behind a router: port forwarding + `ufw` on the host.

## 3. If this is a new box: redeploy

Follow `README-onprem.md` steps 1-6 fresh on the new machine, reusing the same `~/ess-values/hostnames.yaml` (unchanged, since `serverName` is unchanged) and restoring data before you deploy, not after:

- Restore the Postgres dump into the new Postgres pod (or point `postgres.yaml` at your externally-managed instance if you're already doing that).
- Restore the `ess-generated` secret (`kubectl apply -f ess-generated-secret.yaml -n ess`) **before** the first `helm install` — this keeps existing user accounts and device sessions valid instead of the chart minting fresh credentials.
- Restore the `ess-synapse-media` PVC contents (rsync the backed-up media directory onto the new box's volume, or restore the NFS/S3 target if you're using off-box media storage).

Start TLS on `letsencrypt-staging` again for the first deploy on the new box, same reasoning as a fresh install — you're proving the HTTP-01 challenge reaches the new IP before spending a real Let's Encrypt request. Switch to `letsencrypt-prod` once staging certs come back `READY True`.

## 4. If it's the same box, just a new IP: nothing to redeploy

K3s/Traefik/cert-manager don't care what the public IP is — they listen on the box's own interface, not a specific address. Once DNS resolves to the new IP and the port paths are open, existing certs keep working until their normal renewal, and cert-manager will renew fine against the new IP since Let's Encrypt just re-runs the HTTP-01 challenge against whatever the hostname currently resolves to.

Nothing to run here beyond steps 1-2. If you want to force a check rather than wait for the automatic renewal window:

```bash
kubectl get certificate -n ess
kubectl describe certificate -n ess     # confirm it's not stuck mid-challenge
```

## 5. Verify

Same checks as a fresh deploy:

```bash
curl https://example.com/.well-known/matrix/client
curl https://example.com/.well-known/matrix/server
```

Federation: `https://federationtester.matrix.org/api/report?server_name=example.com`, check `FederationOK` is `true`.

Login: open `https://element.example.com`, log in with an existing account. If you restored the `ess-generated` secret and media PVC correctly, this should just work — same accounts, same room history, no re-registration needed.

## Gotchas

- **Elastic IP / static IP**, if your cloud provider offers one, avoids this entire doc going forward. Worth attaching one after the first time you have to do this manually.
- **DNS TTL**: set it low (300s or less) on records you expect might move again, so the next IP change doesn't mean a long wait for propagation.
- **Old IP still in someone's DNS cache**: federation from servers that cached the old IP will fail until their cache expires too — this isn't something you control, it clears itself out on the old TTL.
- **Don't skip the staging cert step on a new box** even though it feels redundant when it "worked before" — it's a new box hitting Let's Encrypt for the first time, and the same debugging value applies as a first-ever deploy.
