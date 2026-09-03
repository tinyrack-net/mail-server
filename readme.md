<div align="center">

# Mail Server

**A Flux GitOps repository for the personal mail Kubernetes cluster.**

[GitOps](#gitops) · [Services](#services) · [Disaster Recovery](#disaster-recovery) · [Bootstrap](#bootstrap)

</div>

---

This repository manages the desired state of the `mail-server` Kubernetes cluster.

It runs Flux on K3s and uses the manifests under `apps` and `infrastructure` to declaratively manage Stalwart Mail, certificates, storage, databases, and cluster maintenance.

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
- [Bulwark Webmail](https://bulwarkmail.org/): JMAP webmail client for `webmail.winetree94.com`.
- [Traefik](https://traefik.io/): HTTP ingress for the Stalwart web/admin surface.
- [CloudNativePG](https://cloudnative-pg.io/): PostgreSQL operator for application data.
- [Longhorn](https://longhorn.io/): persistent volume management and volume backups.
- [cert-manager](https://cert-manager.io/): TLS certificate automation.

Exposed mail protocols are managed through the `stalwart-mail-service` LoadBalancer service:

- SMTP: `25`, `465`, `587`
- IMAP: `143`, `993`
- POP3: `110`, `995`
- ManageSieve: `4190`

## Cilium and mail egress

- K3s uses Cilium `1.20.1` with Pod CIDR `10.57.0.0/16`.
- Ansible bootstraps Cilium before Flux is available; afterward, Flux owns the
  release lifecycle through the `cilium` HelmRelease.
- Both paths consume `infrastructure/base/cilium/values.yaml` as the single
  source of Cilium values.
- Stalwart external IPv4 traffic leaves through Floating IP `46.225.250.229`.
- The A record for `mail.winetree94.com` and the Floating IP PTR must match.
- Use `make preflight`, `make check`, `make apply`, and `make verify` for normal management.
- In Stalwart, route local domains to `local`, external domains to IPv4-only `mx`, and use `mail.winetree94.com` as the EHLO hostname.

## Disaster Recovery

The recovery goal is to install K3s and bootstrap Cilium on a new node, restore
the Sealed Secrets key, and let Flux recreate the cluster state from this
repository.

1. Configure the Floating IP and install K3s with the same cluster and service CIDRs.
2. Bootstrap Cilium from the canonical Helm values.
3. Restore the Sealed Secrets private key before applying encrypted manifests.
4. Bootstrap Flux from `clusters/production`; Flux adopts the existing Cilium release.
5. Wait for `infrastructure` to become ready, then verify `apps` reconciliation.
6. Restore required data from Longhorn backups or application-specific backups.
7. Verify mail delivery, TLS, DNS records, and Stalwart web/admin access.

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

### Hetzner Floating IP

Ansible persistently configures `46.225.250.229/32` as a secondary address on
`eth0` using `/etc/netplan/60-floating-ip.yaml`. The primary address, default
route, DHCP, and IPv6 configuration remain managed by Hetzner cloud-init.

### K3s and Sealed Secrets key

Store the become password and the private key matching
`tinyrack-production-key.crt` in the Ansible Vault, then let Ansible prepare
everything up to the Flux boundary:

```bash
cd ansible
make vault-edit
make preflight
make check
make apply
make apply
make verify
cd ..
```

The first apply configures the Floating IP, normalizes the host packages and
K3s configuration, bootstraps Cilium when Flux is absent, and may restart K3s
once. If the Flux Cilium HelmRelease already exists, Ansible only waits for it
to become Ready. The second apply must report no changes.
Ansible refuses to overwrite an existing Sealed Secrets recovery key when it
does not match the Vault values.

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
