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
| `platform/` | cluster-wide resources (Let's Encrypt issuer) |
| `envs/prod/*-values.yaml` | prod overrides for the app repo charts |
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

## Local rehearsal

Same bootstrap on k3d:
```sh
k3d cluster create dsn -p "8081:80@loadbalancer"
```
then steps 3-4. The app is served on http://localhost:8081.
