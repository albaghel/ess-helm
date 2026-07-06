# Deploy Decisions Log

Track choices made. Fill before Phase 2.

## Server name
- Value: _______________
- Confirmed: [ ]
- NOTE: Cannot change post-deploy without DB wipe.

## Target infra
- [ ] K3s on new VPS
- [ ] Existing K8s cluster (name: ___________)
- [ ] Cloud managed K8s (provider: _________)
- Node IP: _______________

## TLS
- [ ] Let's Encrypt (prod)
- [ ] Let's Encrypt (staging — test first)
- [ ] Wildcard cert file
- [ ] External reverse proxy (nginx/caddy/etc)

## PostgreSQL
- [ ] Bundled (chart-managed) — OK for first deploy / testing
- [ ] External — required for production
  - Host: _______________
  - DB name: _______________

## DNS provider
- Provider: _______________
- TTL to set: 300s (low, during setup)

## Helm release name
- Default: `ess` (used in kubectl commands)
- Namespace: `ess`
