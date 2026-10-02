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
| `bootstrap/root-local.yaml`, `apps-local/` | the same for a local k3d cluster: infra, services and observability |
| `platform/` | cluster-wide resources (Let's Encrypt issuer) |
| `envs/prod/*-values.yaml` | prod overrides for the app repo charts |
| `envs/local/*-values.yaml` | local overrides, layered on top of prod's |
| `envs/prod/secrets/` | SealedSecrets, safe to commit |
| `sealed-secrets/pub-cert.pem` | public cert for sealing |

Sync order (waves): sealed-secrets, cert-manager -> issuer, secrets -> infra -> services.

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
3. Sealing key, before anything syncs (otherwise the controller generates its own and existing secrets won't decrypt):
   ```sh
   kubectl -n kube-system create secret tls sealed-secrets-key --cert=tls.crt --key=tls.key
   kubectl -n kube-system label secret sealed-secrets-key sealedsecrets.bitnami.com/sealed-secrets-key=active
   ```
4. Argo CD and the root app:
   ```sh
   kubectl create namespace argocd
   kubectl apply -n argocd --server-side -f https://raw.githubusercontent.com/argoproj/argo-cd/v3.5.3/manifests/install.yaml
   kubectl apply -f bootstrap/root.yaml
   ```
5. DNS: point a hostname at the VM, set `ingress.host` and `ingress.tls: true` in `envs/prod/services-values.yaml`.

Argo CD UI (not exposed publicly):
```sh
kubectl -n argocd port-forward svc/argocd-server 8443:443
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d
```

## Secrets

```sh
kubectl create secret generic <name> -n dsn --from-literal=KEY=value --dry-run=client -o yaml \
  | kubeseal --cert sealed-secrets/pub-cert.pem --format yaml > envs/prod/secrets/<name>.yaml
```

## Local cluster

A k3d cluster synced from this repo, running the same commit as prod:
```sh
make cluster-up     # in the app repo: k3d, Argo CD, bootstrap/root-local.yaml
make forward        # compose's localhost ports + Argo CD on https://localhost:8443
make cluster-down
```
`apps-local/` deploys infra, services and observability (Grafana, Prometheus, Loki, Jaeger, Redpanda Console on `<name>.localhost:8081`), with `envs/local` values on top of `envs/prod`: plain dev secrets instead of SealedSecrets (no sealing key needed), Keycloak redirects to http://localhost:8081, no TLS. The publish workflow bumps `apps-local/` along with `apps/`. The app is served on http://localhost:8081.
