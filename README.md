# distributed-social-network-gitops

Deployment state for [distributed-social-network](https://github.com/vahan-sahakyan/distributed-social-network), synced to the cluster by Argo CD.

## Flow

```
merge to main (app repo)
  -> publish workflow builds images: ghcr.io/vahan-sahakyan/distributed-social-network/<svc>:<sha>
  -> commits "[deploy] prod <sha>" here: image tag + chart revision
  -> Argo CD syncs the cluster
```

Rollback = `git revert` the deploy commit.

## Layout

| Path | Contents |
|---|---|
| `bootstrap/root.yaml` | app of apps, the only manifest applied by hand |
| `apps/` | one Argo CD Application per component |
| `bootstrap/root-local.yaml`, `apps-local/` | the same for a local k3d cluster, plus observability |
| `platform/` | cluster-wide resources (Let's Encrypt issuer) |
| `platform-local/` | the local cluster's issuer (`local-ca`, signing with the machine's CA) |
| `platform-k3s/` | Traefik's Gateway API provider, both envs |
| `platform-secrets/` | the `openbao` ClusterSecretStore, both envs |
| `envs/prod/*-values.yaml` | prod overrides for the app repo charts |
| `envs/local/*-values.yaml` | local overrides, layered on top of prod's |
| `envs/prod/openbao-values.yaml` | OpenBao, both envs: static seal, self-init (kv mount, auth, secrets) |
| `envs/local/openbao-seed.env` | local dev secret values, public on purpose |

Sync order (waves): cert-manager, OpenBao, External Secrets -> issuer, secret store, Traefik -> infra -> services.

## Bootstrap

Target: one Oracle Cloud Always Free Ampere A1 VM (4 OCPU, 24 GB, Ubuntu 24.04 aarch64).

1. Open 80/443 in the VCN security list, and on the VM:
   ```sh
   sudo iptables -I INPUT 6 -p tcp -m multiport --dports 80,443 -j ACCEPT && sudo netfilter-persistent save
   ```
2. k3s:
   ```sh
   curl -sfL https://get.k3s.io | sh -
   ```
3. OpenBao's unseal key and the first-start secret values, before anything syncs. Keep `openbao-unseal.key` offline: without it the data can't be unsealed. The seed file has the keys of `envs/local/openbao-seed.env` with real values:
   ```sh
   kubectl create namespace openbao
   kubectl -n openbao create secret generic openbao-unseal --from-file=key=openbao-unseal.key   # openssl rand -out openbao-unseal.key 32
   kubectl -n openbao create secret generic openbao-seed --from-env-file=prod-secrets.env
   ```
   OpenBao configures itself from the seed on its first start (`envs/prod/openbao-values.yaml`); delete `openbao-seed` once `openbao-0` is ready.
4. Argo CD and the root app:
   ```sh
   kubectl create namespace argocd
   kubectl apply -n argocd --server-side -f https://raw.githubusercontent.com/argoproj/argo-cd/v3.5.3/manifests/install.yaml
   kubectl apply -f bootstrap/root.yaml
   ```
5. DNS: point a hostname at the VM, set `route.host`, `route.tls: true` and `publicUrl` in `envs/prod/services-values.yaml`; cert-manager issues the certificate through the `dsn` Gateway.

Argo CD UI (not exposed publicly):
```sh
kubectl -n argocd port-forward svc/argocd-server 9443:443
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d
```

## Secrets

Values live in OpenBao (kv v2 mount `dsn`: `postgres`, `minio`, `keycloak-admin`, `grafana-admin`). External Secrets reads them through the `openbao` ClusterSecretStore (Kubernetes auth, read-only policy) and the charts' `ExternalSecret`s build the Secrets the pods use, e.g. `DATABASE_URL` from `postgres`. Nothing secret is in git except the public local dev values.

Changing a value (the `admin` login is created by self-init from `BAO_ADMIN_PASSWORD`):
```sh
kubectl -n openbao exec -it openbao-0 -- sh -c 'bao login -method=userpass username=admin && bao kv patch dsn/minio password=...'
```
External Secrets picks it up within an hour (`refreshInterval`). It only changes the Secret: rotating a password the data store already uses (Postgres, MinIO) also needs the store updated.

Self-init runs once, on empty storage. If it fails (e.g. a missing seed key), fix the seed, delete the `data-openbao-0` PVC and the pod.

## Local cluster

A k3d cluster synced from this repo, running the same commit as prod:
```sh
make cluster-up     # in the app repo: k3d, Argo CD, bootstrap/root-local.yaml
make forward        # compose's localhost ports + Argo CD on https://localhost:9443
make cluster-down
```
`apps-local/` deploys cert-manager, infra, services and observability (Grafana, Prometheus, Loki, Jaeger, Redpanda Console on `https://<name>.localhost:8443`), with `envs/local` values on top of `envs/prod`: certificates from the `local-ca` issuer instead of Let's Encrypt. `make cluster-up` does prod's bootstrap step 3 with local inputs: the machine's CA (generated once in `~/.config/dsn/`) as `dsn-local-ca`, a fresh OpenBao unseal key, and `envs/local/openbao-seed.env` as the seed. Secrets then flow exactly as in prod. The publish workflow bumps `apps-local/` along with `apps/`. The app is served on https://localhost:8443; http://localhost:8081 redirects.
