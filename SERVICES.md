# services used here

what the matrix-stack chart actually deploys, and what it costs.

## deployed by the chart

- **synapse** — the matrix homeserver, does the actual chat/federation work. free, agpl-3.0.
- **matrix authentication service (mas)** — oidc auth/identity in front of synapse. free, open source.
- **element web** — the web chat client. free.
- **element admin** — admin console. free.
- **matrix rtc** — voice/video calls. livekit-server does the actual media routing (sfu), lk-jwt-service hands out join tokens. both free/open source, self-hosted. livekit also sells a hosted cloud version but that's not what's running here.
- **haproxy** — sits in front of everything, routes ingress traffic. free.
- **matrix hookshot** — bridges rooms to github/gitlab/jira/webhooks. free.
- **postgresql** — the database, backs synapse and mas. free.
- **redis** — caching/pubsub for synapse. free, bsd-3.
- **matrix-tools** — helper/init container this project builds itself (secret gen etc). free.
- **postgres-exporter / redis-exporter** — optional prometheus sidecars. free.
- **cert-manager** — optional, handles tls certs automatically. free itself; if you point it at let's encrypt that's free too, a commercial CA obviously isn't.

## stuff you need around the chart but it doesn't deploy

- **docker desktop** — free for personal use and small business, but once a company crosses 250 employees or $10M revenue docker wants you on a paid business plan.
- **k3d** — free, only really used for local/CI clusters anyway.
- **kubernetes** itself is free/open source, but if you're on a managed offering (eks/gke/aks) that's billed per node/cluster by the cloud provider.
- **helm, uv, yq** — all free cli tooling.
- **ghcr.io** — hosts most of the images here (element-hq/*, matrix-hookshot). free for public images.
- **docker hub** — hosts postgres, redis, haproxy, livekit-server, the exporters. free to pull publicly, but there are rate limits, and private repos cost money.
- **oci.element.io** — element's own registry for synapse/element web/admin/mas builds. free.
- **github actions** — runs the ci in .github/workflows. free on public repos, costs money on private repos past the included minutes.
- **cloud compute** (aws ec2 or whatever) — only comes into play once you're not running this locally. billed by the hour by whoever's hosting it, no way around that.

basically nothing in the chart itself has a license fee attached — it's all self-hosted open source. the costs show up in where you run it: compute, managed k8s, a paid CA if policy demands it, or blowing past free-tier limits on actions/docker hub.

## running this for a company

worth knowing: this chart is **ess community**, and element's own docs cap that at "small/mid-scale, non-commercial, up to 100 users." past that you're supposed to be on **ess pro** (element's paid tier — adds proper iam, ha, multi-tenancy, compliance stuff, support contracts). there's also **ess ti-m**, a pro variant for german health-sector compliance (gematik ti-messenger/epa) if that's relevant.

rough guide:
- under ~100 people, non-commercial-ish use — community edition as-is is genuinely fine, free.
- past 100, or you actually need enterprise features/support — budget for ess pro.
- regulated healthcare context in germany — ess ti-m.

things that quietly start costing money once a company (not a solo dev) runs this at real scale: docker desktop past the 250-employee/$10M threshold, github actions minutes on a private repo, docker hub pull limits under heavy ci/employee use, and obviously cloud compute which was never free to begin with and scales with headcount regardless of which ess edition you pick.

## can calls scale?

matrix rtc scaling is controlled by two replica counts in values.yaml — matrixRTC.sfu.replicas and matrixRTC.authorisationService.replicas, both default to 1. there's no autoscaler wired up for either (checked templates/matrix-rtc, no HPA in there), so it's manual. the resources.requests/limits knobs are there too, so vertical scaling (bigger pod, more cpu/bandwidth) works fine out of the box.

horizontal scaling doesn't really work as shipped though. bumping sfu.replicas past 1 just gives you multiple independent sfu pods with no coordination between them — livekit's actual multi-node mode needs a shared redis backend for room/node discovery, and this chart's sfu config (configs/matrix-rtc/sfu/config-overrides.yaml.tpl) doesn't set that up. so you'd end up with calls potentially split across sfus that don't know about each other, which doesn't work.

so: vertical scaling, yes, easily. real horizontal scale-out for lots of concurrent large calls would mean wiring up livekit's redis-backed clustering yourself — not something this chart does for you — or moving to ess pro / livekit cloud where someone else manages that.
