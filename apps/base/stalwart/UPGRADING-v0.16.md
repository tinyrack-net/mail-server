# Stalwart v0.16 Migration Runbook

This repository is prepared for Stalwart `v0.16.8`, but production must not be rolled forward until the offline migration steps have completed. Stalwart 0.16 replaces the old `config.toml` management model with a small `config.json` plus JMAP-managed objects applied with `stalwart-cli`.

## Prepared Manifest Changes

- `stalwart.deployment.yaml` now uses `stalwartlabs/stalwart:v0.16.8`.
- The startup config is mounted at `/etc/stalwart/config.json`.
- An initContainer renders `config.json` from `stalwart-database-user-secret`; Stalwart 0.16 does not expand `%{env:...}%` inside `config.json`.
- `stalwart.config.yaml` is kept as a non-secret reference for the PostgreSQL `DataStore` shape.
- `STALWART_PUBLIC_URL` is set to `https://mail.winetree94.com` for reverse-proxy/OAuth discovery.

## Do Not Skip

Generated migration files can contain secrets, password hashes, DKIM material, TLS private keys, and live routing credentials. Keep them outside Git, for example under `stalwart-v0.16-migration/`, which is ignored by `.gitignore`.

## Preflight

1. Suspend Flux before applying these manifests to production:

   ```bash
   flux suspend kustomization stalwart -n flux-system
   ```

2. Download the official migration script:

   ```bash
   mkdir -p stalwart-v0.16-migration
   curl -fL -o stalwart-v0.16-migration/migrate_v016.py \
     https://raw.githubusercontent.com/stalwartlabs/stalwart/refs/heads/main/resources/scripts/migrate_v016.py
   ```

3. Create a Python environment for the migration script:

   ```bash
   python3 -m venv stalwart-v0.16-migration/.venv
   stalwart-v0.16-migration/.venv/bin/pip install requests urllib3
   ```

4. Rehearse the dump and conversion against the running 0.15.5 server before the maintenance window. Use admin credentials that can read settings and principals:

   ```bash
   stalwart-v0.16-migration/.venv/bin/python stalwart-v0.16-migration/migrate_v016.py dump \
     --url https://mail.winetree94.com \
     --username '<admin-user>' \
     --password '<admin-password>' \
     --settings stalwart-v0.16-migration/settings.json \
     --principals stalwart-v0.16-migration/principals.json

   stalwart-v0.16-migration/.venv/bin/python stalwart-v0.16-migration/migrate_v016.py convert \
     --settings stalwart-v0.16-migration/settings.json \
     --principals stalwart-v0.16-migration/principals.json \
     --config stalwart-v0.16-migration/config.json \
     --output stalwart-v0.16-migration/export.json
   ```

5. Compare `stalwart-v0.16-migration/config.json` with `stalwart.config.yaml`. If the generated DataStore differs, update the ConfigMap before starting the 0.16 pod.

6. Build a throwaway 0.16 instance and recreate the settings the migration script does not convert. Export them with `stalwart-cli snapshot` and keep the resulting NDJSON file with the migration artifacts.

Required manual settings for this deployment:

- Network listeners for `80`, `443`, `25`, `587`, `465`, `143`, `993`, `110`, `995`, and `4190`.
- Traefik/reverse-proxy forwarded-header behavior.
- SMTP2GO relay route for non-local mail.
- S3 blob store `tinyrack-stalwart` with prefix `stalwart/`.
- stdout logging/telemetry preferences.
- Any spam, rate-limit, policy, or retention settings required after review.

## Maintenance Window

1. Freeze admin, domain, account, and routing changes.

2. Run a final 0.15 dump and conversion with the same commands from preflight.

3. Stop Stalwart:

   ```bash
   kubectl -n stalwart-system scale deploy/stalwart-deployment --replicas=0
   kubectl -n stalwart-system get pods -l app=stalwart
   ```

4. Take an explicit CloudNativePG backup after the pod is stopped. Record the backup name before continuing.

5. Apply the prepared v0.16 manifests while Flux remains suspended.

6. Start v0.16 in recovery mode. Prefer a temporary imperative patch during the window rather than committing recovery mode to Git:

   ```bash
   kubectl -n stalwart-system set env deploy/stalwart-deployment \
     STALWART_RECOVERY_MODE=1 \
     STALWART_RECOVERY_ADMIN='admin:<temporary-password>'
   ```

7. Port-forward the recovery endpoint. Recovery mode listens on `8080` by default:

   ```bash
   kubectl -n stalwart-system port-forward deploy/stalwart-deployment 8080:8080
   ```

8. Apply the converted objects:

   ```bash
   export STALWART_URL=http://127.0.0.1:8080
   export STALWART_USER=admin
   export STALWART_PASSWORD='<temporary-password>'
   stalwart-cli apply --file stalwart-v0.16-migration/export.json
   ```

9. Apply the throwaway/test 0.16 snapshot containing manually recreated listeners, routing, S3 blob store, proxy behavior, and telemetry:

   ```bash
   stalwart-cli apply --file stalwart-v0.16-migration/test-snapshot.json
   ```

10. Remove recovery mode and restart normally:

    ```bash
    kubectl -n stalwart-system set env deploy/stalwart-deployment \
      STALWART_RECOVERY_MODE- \
      STALWART_RECOVERY_ADMIN-
    kubectl -n stalwart-system rollout restart deploy/stalwart-deployment
    ```

11. Log in to `https://mail.winetree94.com/admin`, create or verify a permanent administrator, and trigger disk quota recalculation from the task panel.

12. Resume Flux only after the live cluster state matches Git:

    ```bash
    flux resume kustomization stalwart -n flux-system
    ```

## Verification

```bash
kubectl -n stalwart-system get deploy stalwart-deployment
kubectl -n stalwart-system get pods -l app=stalwart
kubectl -n stalwart-system logs deploy/stalwart-deployment
flux get kustomization stalwart -n flux-system
```

Verify these externally:

- `https://mail.winetree94.com/admin` loads and redirects to `https://mail.winetree94.com`, not `:8080` or a pod IP.
- Permanent admin login works and recovery admin is removed.
- SMTP inbound on `25` works.
- Authenticated submission on `587` and `465` works.
- IMAP on `143` and `993` works.
- POP3 on `110` and `995` works if clients use it.
- ManageSieve on `4190` works if clients use it.
- Existing mailbox contents are visible.
- Outbound non-local delivery goes through SMTP2GO.
- S3 blob reads and writes work.
- Disk quota recalculation finishes.

## Rollback

Before recovery mode has mutated PostgreSQL, revert the manifests to 0.15.5 and scale the deployment back up.

After recovery mode has run, do not start 0.15.5 against the mutated database. Rollback requires stopping Stalwart, restoring PostgreSQL from the explicit pre-migration CloudNativePG backup, reverting to the old 0.15.5 `config.toml` manifests, and starting 0.15.5 again. Leave the S3 blob bucket untouched during rollback.
