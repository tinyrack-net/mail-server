<div align="center">

# Mail Server

**A Flux GitOps repository for the personal mail Kubernetes cluster.**

[GitOps](#gitops) · [Services](#services) · [Disaster Recovery](#disaster-recovery) · [Bootstrap](#bootstrap)

</div>

---

This repository manages the desired state of the `mail-server` Kubernetes cluster.

It runs Flux on K3s and uses the manifests under `apps` and `infrastructure` to declaratively manage Stalwart Mail, certificates, storage, databases, observability, and cluster maintenance.

## GitOps

- `clusters/production` is the Flux bootstrap path.
- `clusters/production/infrastructure.yaml` syncs `./infrastructure/overlays/production`.
- `clusters/production/apps.yaml` syncs `./apps/overlays/production` and depends on `infrastructure`.
- `infrastructure/overlays/production/kustomization.yaml` is the infrastructure enable/disable list.
- `apps/overlays/production/kustomization.yaml` is the application enable/disable list.
- `apps/base/*` and `infrastructure/base/*` hold the workload manifests.
- Secrets are committed only as encrypted Sealed Secrets manifests.

## Services

- [Stalwart Mail](https://stalw.art/): mail server for `mail.winetree94.com`.
- [Traefik](https://traefik.io/): HTTP ingress for the Stalwart web/admin surface.
- [CloudNativePG](https://cloudnative-pg.io/): PostgreSQL operator for application data.
- [Longhorn](https://longhorn.io/): persistent volume management and volume backups.
- [cert-manager](https://cert-manager.io/): TLS certificate automation.
- [kube-prometheus-stack](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack), [Loki](https://grafana.com/oss/loki/), and [Alloy](https://grafana.com/oss/alloy-opentelemetry-collector/): observability.

Exposed mail protocols are managed through the `stalwart-mail-service` LoadBalancer service:

- SMTP: `25`, `465`, `587`
- IMAP: `143`, `993`
- POP3: `110`, `995`
- ManageSieve: `4190`

## Disaster Recovery

The recovery goal is to install K3s on a new node, restore the Sealed Secrets key, and let Flux recreate the cluster state from this repository.

1. Install K3s with the same cluster and service CIDRs.
2. Restore the Sealed Secrets private key before applying encrypted manifests.
3. Bootstrap Flux from `clusters/production`.
4. Wait for `infrastructure` to become ready, then verify `apps` reconciliation.
5. Restore required data from Longhorn backups or application-specific backups.
6. Verify mail delivery, TLS, DNS records, and Stalwart web/admin access.

DR guidelines:

- Git is the source of truth for declarative infrastructure.
- Keep the Sealed Secrets private key backed up separately and securely.
- Data volumes are not restored from Git; verify backup policy per service.
- Database-like volumes should use their own backup/restore flow rather than generic Longhorn volume backups when applicable.
- After recovery, verify Flux, Sealed Secrets, certificates, storage, database, ingress, and mail protocols in that order.

## Bootstrap

### Tailscale

```bash
sudo tailscale up \
  --accept-dns=false \
  --reset
```

### K3s

```bash
curl -fL https://get.k3s.io | \
sh -s - server \
  --cluster-init \
  --cluster-cidr=10.57.0.0/16 \
  --service-cidr=10.58.0.0/16 \
  --tls-san=100.127.220.52 \
  --tls-san=mail-server.time-inconnu.ts.net
```

### Sealed Secrets key

```bash
kubectl create namespace sealed-secrets
kubectl -n sealed-secrets apply -f ./main.key.yaml --force
```

> The committed certificate is public. This restore step needs the private key backup.

### Flux

```bash
flux bootstrap github \
  --repository=mail-server \
  --branch=main \
  --path=./clusters/production \
  --owner=tinyrack-net
```

## Sealed Secrets

Create Kubernetes Secrets locally and seal them before committing:

```bash
kubectl create secret generic some-secret \
  --namespace some-namespace \
  --dry-run=client \
  --from-literal=SOME_SECRET_KEY=SOME_SECRET_VALUE \
  -o yaml | \
  kubeseal --cert ./tinyrack-production-key.crt \
  > ./some.secret.yaml
```
