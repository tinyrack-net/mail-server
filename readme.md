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

- [Stalwart Mail](https://stalw.art/) 1.x: mail server for `mail.winetree94.com`.
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

## Stalwart storage and OAuth clients

- Stalwart 1.0 uses the `stalwart_v1` PostgreSQL database in
  `stalwart-database-cluster`; `stalwart-v1.config.json` is its data store
  definition. The 0.16 `stalwart` database and `stalwart-config` ConfigMap are
  kept only as the migration source until they are retired.
- Message bodies, attachments and application bundles live in the external S3
  blob store configured inside Stalwart (bucket `tinyrack-stalwart`, prefix
  `stalwart/`). Its credentials are stored in the Stalwart settings, not in
  this repository.
- A blob store change does not move existing blobs. Before a future blob store
  change, copy every blob first, or the older messages become unreadable.
- Stalwart 1.0 accepts only registered OAuth clients. Bulwark signs in as the
  public first-party client `bulwark-webmail`, registered in Stalwart with one
  redirect URI per Bulwark locale (`https://webmail.winetree94.com/<locale>/auth/callback`).
  Re-register it after restoring Stalwart settings from scratch.
- Stalwart 1.x upgrades are in place: bump the image tag after reading the
  release notes.

### Retiring the 0.16 migration

`stalwart-migrate` copied 0.16.27 into 1.0.0 on 2026-10-27. Its toolbox
(`stalwart-migrate.yaml`) and state PVC remain only for a rollback. A rollback
after 1.0 has served mail must use `stalwart-migrate rollback` into a new, empty
0.16 store, because the S3 blob store is shared; see the
[rollback guide](https://stalw.art/docs/migration/upgrade-0-16/rollback).

Once 1.0 has proved itself, remove the toolbox and its CNP rules, the
`stalwart-config` ConfigMap, and the AWS environment variables of the Stalwart
Deployment once no Stalwart setting references them. Then delete the state PVC
and drop the `stalwart` database.

## Cilium and mail egress

- K3s uses Cilium `1.20.4` with Pod CIDR `10.57.0.0/16`.
- Ansible bootstraps Cilium before Flux is available; afterward, Flux owns the
  release lifecycle through the `cilium` HelmRelease.
- Both paths consume `infrastructure/base/cilium/values.yaml` as the single
  source of Cilium values.
- Stalwart external IPv4 traffic leaves through the Floating IP configured in the Ansible inventory.
- The A record for `mail.winetree94.com` and the Floating IP PTR must match.
- Use `make preflight`, `make check`, `make apply`, and `make verify` for normal management.
- In Stalwart, route local domains to `local`, external domains to IPv4-only `mx`, and use `mail.winetree94.com` as the EHLO hostname.

## Disaster Recovery

The recovery goal is to install K3s and bootstrap Cilium on a new node, restore
the Sealed Secrets key, and let Flux recreate the cluster state from this
repository.

1. Configure the Floating IP and install K3s with the same cluster and service CIDRs.
2. Bootstrap Cilium from the canonical Helm values.
3. Restore the single `tinyrack-production-key` Sealed Secrets private key before applying encrypted manifests.
4. Bootstrap Flux from `clusters/production`; Flux adopts the existing Cilium release.
5. Wait for `infrastructure` to become ready, then verify `apps` reconciliation.
6. Restore required data from Longhorn backups or application-specific backups.
7. Verify mail delivery, TLS, DNS records, and Stalwart web/admin access.

DR guidelines:

- Git is the source of truth for declarative infrastructure.
- Keep the Ansible Vault password separately and securely; Git contains only the encrypted private key backup.
- The controller must load only `tinyrack-production-key` before Flux applies encrypted manifests.
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

Ansible persistently configures the Floating IP from its inventory as a secondary address on
`eth0` using `/etc/netplan/60-floating-ip.yaml`. The primary address, default
route, DHCP, and IPv6 configuration remain managed by Hetzner cloud-init.

### K3s and Sealed Secrets key

Store the become password and the private key matching
`tinyrack-production-key.crt` in the Ansible Vault. Automatic sealing-key
renewal is disabled, so Ansible restores and verifies exactly this one key:

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
Ansible refuses to overwrite a mismatched recovery key and fails when any
additional active sealing key exists. This detects drift without deleting key
material automatically.

### Host maintenance

Apply pending Ubuntu packages during a maintenance window. The playbook performs
a safe package upgrade, reboots only when required, and waits for the K3s node
to become ready again:

```bash
make -C ansible preflight
make -C ansible upgrade-host
make -C ansible verify
```

### K3s upgrades

K3s follows the **stable** release channel. Patch releases within the current
stable minor are routine; a new Kubernetes minor is adopted only after K3s
promotes it to `stable` and Cilium, Kyverno, and Longhorn support that minor.
Do not merge a K3s minor bump just because it appears upstream.

Check the current stable target before preparing an upgrade:

```bash
curl -s https://update.k3s.io/v1-release/channels |
  jq -r '.data[] | select(.id == "stable") | .latest'
```

Upgrade procedure:

1. Confirm Flux Kustomizations and HelmReleases, CloudNativePG, and Longhorn are
   healthy, and record the current node, Plan, and Ansible versions.
2. Create an etcd snapshot and a fresh CNPG backup.
3. Update the Ansible pin and the SUC Plan together:
   - `ansible/roles/setup_k3s_server/defaults/main.yml`
   - `infrastructure/base/k3s-upgrade/k3s-server-plan.yaml`
4. Merge during a maintenance window. Flux applies the Plan; the
   system-upgrade-controller cordons the node and runs the upgrade job. Do not
   run `make -C ansible apply` while that job is running.
5. Watch the Plan, node, and pods until the Plan reports `Complete`.
6. Verify `make -C ansible verify`, a no-change `make -C ansible check`, Flux
   reconciliation, Cilium, mail protocols, CloudNativePG, and Longhorn.
7. Roll back by restoring the pre-upgrade etcd snapshot. The
   system-upgrade-controller does not downgrade in place.

### Flux

```bash
flux --context mail-server bootstrap github \
  --repository=mail-server \
  --branch=main \
  --path=./clusters/production \
  --owner=tinyrack-net
```

## Renovate

Dependency updates run through the Renovate GitHub App. An organization owner
must install the app for `tinyrack-net/mail-server` before `renovate.json` can
create Dependency Dashboard entries or pull requests. Minor and major updates
require explicit Dependency Dashboard approval, and no update is auto-merged.
K3s minor updates are approved only after the stable-channel and component
support checks in [K3s upgrades](#k3s-upgrades).

## Configuration files

- Keep a HelmRelease focused on chart lifecycle and load chart configuration
  from a sibling `<component>.values.yaml` through `spec.valuesFrom`.
- Generate Helm values ConfigMaps with a stable name, the
  `reconcile.fluxcd.io/watch: Enabled` label, and `values.yaml` as the data key.
- Keep application-native YAML, TOML, JSON, ENV, and Alloy River configuration
  in files named for the owning component. Use Kustomize's default name hash
  when a Pod directly mounts or imports a ConfigMap so changes roll the Pod.
- Disable the name hash only when the consumer requires a stable name, such as
  Helm values and Alloy's externally managed, dynamically reloaded ConfigMap.
- Keep credentials out of values and configuration files. Secrets remain
  encrypted SealedSecret manifests or references to existing Secrets.

## Sealed Secrets

Create Kubernetes Secrets locally and seal them before committing:

```bash
kubectl --context mail-server create secret generic some-secret \
  --namespace some-namespace \
  --dry-run=client \
  --from-literal=SOME_SECRET_KEY=SOME_SECRET_VALUE \
  -o yaml | \
  kubeseal --cert ./tinyrack-production-key.crt \
  > ./some.secret.yaml
```

Always seal with the committed certificate instead of fetching a certificate
from the live controller. Rotate the sealing key annually, or immediately after
suspected exposure: create and back up the replacement key first, re-seal every
manifest, verify the GitOps rollout, and only then retire the previous key. If
the sealing key may have leaked, rotate the underlying passwords, tokens, and
session secrets as well.
