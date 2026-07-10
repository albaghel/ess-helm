# deploying locally (mac, k3d)

notes on getting ess-helm running on a mac with k3d. follows the DEVELOPERS.md path in this repo, not the older fork guides floating around.

## prereqs

need docker desktop running, plus uv, helm v3, yq, k3d installed. quick check:

```bash
docker info
uv --version
helm version
yq --version
k3d version
```

## python env

```bash
uv sync
source .venv/bin/activate
which pytest   # just to confirm the venv actually activated
```

## cluster

```bash
./scripts/setup_test_cluster.sh
```

spins up a k3d cluster with ingress, cert-manager, and a self-signed CA.

if you've got a stale cluster lying around from a previous attempt (containers showing Exited), blow it away first:

```bash
./scripts/destroy_test_cluster.sh
./scripts/setup_test_cluster.sh
```

then check:

```bash
docker ps -a
kubectl get nodes
```

## trust the cert

setup_test_cluster.sh drops a self-signed CA at ~/.config/ess-helm-ca/ca.crt (CN=ess-ca). trust it or the browser will complain:

```bash
sudo security add-trusted-cert -d -r trustRoot -k /Library/Keychains/System.keychain ~/.config/ess-helm-ca/ca.crt
```

check it landed:

```bash
security find-certificate -c "ess-ca"
```

## deploy

```bash
helm -n ess upgrade -i ess charts/matrix-stack \
  -f charts/matrix-stack/ci/test-cluster-mixin.yaml \
  -f charts/matrix-stack/ci/example-default-enabled-components-values.yaml
```

add local overrides with another -f if you've got charts/matrix-stack/user_values/local.yaml (it's gitignored).

```bash
kubectl get pods -n ess
```

postgres and synapse sometimes sit Pending for a minute on first boot, that's normal, give it a bit.

## verify

- hit https://element.ess.localhost, should load clean, no cert warning
- register a user: `mas-cli manage register-user`
- log into element web with it

if that works you're done.
