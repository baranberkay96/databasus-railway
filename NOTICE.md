# NOTICE

This Railway template is a community package. It is not affiliated with, sponsored by or endorsed by the Databasus project or Rostislav Dugin.

## What this template runs (unmodified upstream image)

| Component | Source | License |
|---|---|---|
| Databasus | `databasus/databasus:v3.61.0` (Docker Hub; identical index digest `sha256:4adcd3fd2ab48168a5ee05eb5dbacf1d179d6df387b0daa1c311f57d4f9dd31e` on `ghcr.io/databasus/databasus:v3.61.0`), built from https://github.com/databasus/databasus tag v3.61.0 | Apache License 2.0 |
| Bundled inside that image | PostgreSQL 17 server (PGDG packages on Debian bookworm), PostgreSQL / MySQL / MariaDB / MongoDB client tools, `gosu`, `goose` | Their own licenses, as shipped by upstream |
| Bucket | Railway Bucket (Railway platform service) | Railway terms |

Upstream copyright and trademark notice (from the repository's `NOTICE.md`, which permits reproducing it):

> Copyright © 2025–2026 Rostislav Dugin and contributors.
> "Databasus" is a trademark of Rostislav Dugin.

The name "Databasus" and the upstream logo (used as the template icon, loaded from the v3.61.0 tag) are used only to identify the software this template deploys, which upstream's notice lists as permitted referential use for deployment templates. The Apache-2.0 license covers the code, not the name or logo.

## What this package adds

No upstream file is copied into this package. The template references the image above and sets environment variables documented by upstream. The start command is original work: it wraps the image's own `/app/start.sh` with a `gosu` shim that calls upstream's documented console command `./main --create-user` before the web server starts, sets the upstream `users_settings.is_allow_external_registrations` flag, and stops the embedded PostgreSQL with `pg_ctl` on shutdown.

## Upstream material checked (Databasus v3.61.0, 2026-10-20)

From `_upstream/databasus__databasus` (Apache-2.0), confirmed identical to tag v3.61.0 for the files below via the GitHub contents API, read for configuration facts only:

- `Dockerfile` (runtime layout, embedded PostgreSQL 17, `/databasus-data` volume, `databasus healthcheck` probe) and `docker/start.sh` (storage preparation, embedded PostgreSQL bootstrap, internal DSN, final `exec gosu databasus ./main`)
- `deploy/helm/values.yaml` and `templates/statefulset.yaml` (production settings: probes on `/api/v1/system/version`, `LOG_FILE_IS_ENABLED=false`, 1 GiB / 0.5 CPU, single replica with a volume)
- `backend/cmd/main.go` (`--create-user`, `--new-password`, `--disable-2fa`, listen address `:4005`), `backend/cmd/healthcheck.go`, `backend/internal/config/config.go` (environment variables)
- `backend/internal/features/users/services/console_account.go`, `user_services.go`, `repositories/users_settings_repository.go`, `migrations/20251031143019_add_many_users_and_settings.sql` (first account becomes administrator; registration default on)
- `backend/internal/features/storages/models/s3/model.go` (custom endpoint, virtual-hosted vs path-style lookup with minio-go v7.2.1)
- `backend/internal/features/restores/usecases/postgresql/restore_backup_uc.go` (streamed vs file-based restores)
- Website docs in `website/app/(en)/`: installation, advanced-config, password, security, faq (physical backups and PITR), restore-verification, notifiers, storages

Also checked: Docker Hub and GHCR tag manifests and the image config for v3.61.0; GitHub release v3.61.0 (published 2026-10-10); `railwayapp-templates/postgres-ssl` `Dockerfile.18` and `wrapper.sh` (pg_hba remote rule, pgBackRest-based PITR); Railway docs on Buckets, Point-in-Time Recovery, volumes, outbound SMTP, private networking and `railway ssh`; Debian bookworm `gosu` file list (`/usr/sbin/gosu`).

## Market templates

The Railway marketplace templates in `_market/` (`databasus`, Railway's `I4zGrH` "Postgres Backup", and a Directus stack that embeds the same cron) were read for their service layout and settings. `_upstream/railwayapp-templates__postgres-s3-backups` has no license and was read for comparison only; nothing from it is reused. No README or description text from any market template is reused.
