# Databasus v3 Postgres Backups and Restore: Railway template spec

## Listing

| Field | Value |
|---|---|
| Name | **Databasus v3 Postgres Backups and Restore** (`v3` = pinned release v3.61.0; the project tags releases with `v`) |
| Description | Scheduled database backups to S3 with a dashboard, alerts and restore. (70 chars) |
| Category | Storage |
| Tags | databasus, backup, postgres, postgresql, mysql, mariadb, mongodb, s3, restore, pg-dump |
| Icon | `https://cdn.jsdelivr.net/gh/databasus/databasus@v3.61.0/frontend/public/logo.svg` (upstream logo pinned to the v3.61.0 tag, 200 OK, image/svg+xml) |

The listing is delivered as written. Databasus backs up more than Postgres (MySQL, MariaDB and MongoDB, logical dumps only); the description already says "database backups" and the README lists every engine, so no wording change was needed. "Restore" means restoring a logical backup into a database you choose (see Restore). The template does **not** claim point-in-time recovery for Railway Postgres (see PITR).

## Base market templates

| Market template | Topology | Kept | Problems fixed |
|---|---|---|---|
| `databasus` "Databasus" (12 active) | One `databasus/databasus:v3.61.0` service, volume `/databasus-data`, domain on 4005, a Railway Bucket | **Base.** Same image and tag, same volume path and port, Bucket included | No variables at all: the first person to open the public URL can sign up and becomes the instance administrator (upstream gives the first account on an empty instance the admin role), and open sign-up stays on afterwards; no healthcheck, no `PORT`; the Bucket is created but nothing tells you how to connect it (Databasus defaults to path-style requests, Railway Buckets need virtual-hosted style); the embedded PostgreSQL is killed on every redeploy; README makes generic claims and has no Railway-specific steps |
| `I4zGrH` "Postgres Backup" (Railway, 257 active) | `railwayapp-templates/postgres-s3-backups` cron service (node + `pg_dump`), Bucket | The idea of a Railway Bucket as the default target with references to its credentials | Postgres only; `PG_VERSION=17` client while new Railway Postgres services run 18 (`pg_dump` refuses to dump a newer server major); no retention, so the bucket grows forever; no restore path, no alerts, no encryption, no UI; dump is written to the container disk before upload. Not reused (no license), only used for comparison |
| `directus-with-postgres-redis-and-auto-ba` | Directus stack that embeds the Railway cron above | Nothing | Unrelated app; listed in `_market` because it contains the same backup cron |

Upstream used for everything else: `_upstream/databasus__databasus` (Apache-2.0). It is identical to tag v3.61.0 for every file the template depends on (`docker/start.sh`, `Dockerfile`, `backend/cmd/main.go`, `backend/cmd/healthcheck.go`, the user and S3 storage code), checked with `gh api repos/databasus/databasus/contents/...?ref=v3.61.0`. No market README text was reused.

## Pinned versions

| Service | Image | Version | Released | Checked |
|---|---|---|---|---|
| Databasus | `databasus/databasus:v3.61.0` (Docker Hub; same index digest on `ghcr.io/databasus/databasus:v3.61.0`) | v3.61.0, index digest `sha256:4adcd3fd2ab48168a5ee05eb5dbacf1d179d6df387b0daa1c311f57d4f9dd31e`, linux/amd64 (312 MB compressed) and linux/arm64 (294 MB) | 2026-10-10 (GitHub release 19:44 UTC, image built 19:39 UTC) | 2026-10-20 |
| Bucket | Railway Bucket (S3-compatible, Tigris) | n/a | n/a | 2026-10-20 |

Image config read from the registry (no pull): `ENTRYPOINT ["/app/start.sh"]`, no CMD, `WORKDIR /app`, `ENV_MODE=production`, `APP_VERSION=3.61.0`, exposed 4005, `VOLUME /databasus-data`, runs as root until `start.sh` drops to the `databasus` user with `gosu`.

## Architecture

```
 Browser ── HTTPS ──► Databasus :4005 (public domain, sign-in required)
                     │  Go backend + embedded web UI
                     │  embedded PostgreSQL 17 on localhost:5437 (metadata, encrypted credentials)
                     │  volume /databasus-data: pgdata/, secret.key, temp/, backups/ (local storage)
                     │
                     ├── private network ──► your databases (pg_dump / mysqldump / mongodump, read-only user)
                     │                       e.g. postgres.railway.internal:5432
                     ├── HTTPS (S3 API) ───► Railway Bucket (backup files, encrypted by Databasus)
                     └── HTTPS ────────────► Slack / Telegram / Discord / Teams / Mattermost / webhook alerts
```

Databasus is one container by design: upstream's `start.sh` always initializes and runs its own PostgreSQL 17 cluster under `/databasus-data/pgdata`, even when `DATABASE_DSN` points elsewhere. A separate Railway Postgres for metadata would therefore run a second, unused Postgres server, so the template keeps upstream's embedded storage on the service volume (same as upstream's Docker and Helm installs).

## Services

### Databasus

| Setting | Value |
|---|---|
| Source | image `databasus/databasus:v3.61.0` |
| Start | `/bin/bash -c '...'` admin bootstrap wrapper around the image's `/app/start.sh` (below) |
| Healthcheck | `/api/v1/system/version` (the dependency-free endpoint upstream uses for its Docker and Kubernetes probes; it answers only after migrations and the admin bootstrap), timeout 600 s |
| Public domain | yes, port 4005 (`PORT=4005`, Rule 8.1; Go listens on `:4005`, dual-stack) |
| Restart | ON_FAILURE, 10 retries |
| Volume | `/databasus-data` (embedded PostgreSQL, `secret.key`, temp files, optional local backups) |
| Draining | `RAILWAY_DEPLOYMENT_DRAINING_SECONDS=60` |

Variables (all have descriptions in `template.json`):

| Variable | Default | Purpose |
|---|---|---|
| `DATABASUS_ADMIN_EMAIL` | empty, **required** | Login of the administrator created on first boot |
| `DATABASUS_ADMIN_PASSWORD` | `${{secret(24, ...)}}` | Initial administrator password (first boot only) |
| `DATABASUS_DISABLE_SIGNUP` | `true` | Turns off "Allow external registrations" on first boot |
| `DATABASUS_URL` | `https://${{RAILWAY_PUBLIC_DOMAIN}}` | Base URL in emails, SMTP HELO name |
| `LOG_FILE_IS_ENABLED` | `false` | stdout only, as upstream's Helm chart does |
| `LOG_LEVEL` | `info` | |
| `IS_DISABLE_ANONYMOUS_TELEMETRY` | `false` | Upstream default (anonymous telemetry on); `true` opts out |
| `SMTP_HOST`, `SMTP_PORT`, `SMTP_SECURITY`, `SMTP_USER`, `SMTP_PASSWORD`, `SMTP_FROM` | empty | Optional instance mail (invites, password reset, sign-in codes); outbound SMTP needs Railway Pro |
| `S3_ENDPOINT`, `S3_BUCKET`, `S3_REGION`, `S3_ACCESS_KEY_ID`, `S3_SECRET_ACCESS_KEY` | `${{Bucket.*}}` | Copy helpers for the S3 storage form. Databasus does not read them (storages live in its database); names avoid `AWS_*` and `PG*`, which child tools such as `pg_dump` would pick up from the environment |

Deliberately **not** set: `DATABASE_DSN` (would bypass the embedded metadata database), any `PG*` variable (Databasus passes the container environment to `pg_dump`/`pg_restore`, so `PGHOST`, `PGSSLMODE` etc. would leak into every backup job), `PUID`/`PGID` (the volume is root-owned, so upstream's automatic choice of 999 is correct).

### Bucket

Railway Bucket named `Bucket`. Databasus stores a small backup as one object and a larger one as numbered part objects plus a `.parts` manifest (uploaded in 16 MiB multipart chunks, so memory stays flat), and deletes expired backups itself (Railway Buckets have no lifecycle rules). Bucket egress is free; uploads count as Databasus service egress.

## Start command (admin bootstrap)

Why: upstream has no environment variables for an initial administrator. On an empty instance the **first account that signs up becomes the administrator**, and "Allow external registrations" defaults to on (`users_settings` migration: `DEFAULT TRUE`). On a public Railway domain that is an open first-visitor admin claim. Upstream's supported way to provision an account without the browser is the console command `./main --create-user --email ... --password ... --role=admin`, which is idempotent and, on an empty instance, makes that account the instance administrator (`backend/internal/features/users/services/console_account.go`). It needs the embedded PostgreSQL, which `start.sh` starts just before `exec gosu databasus ./main`.

How: the start command puts a small `gosu` shim first in `PATH` and then runs the unmodified `/app/start.sh`. Every `gosu` call passes straight through to `/usr/sbin/gosu`, except the final `gosu databasus ./main`. There, before the web server exists, the shim:

1. skips everything if `/databasus-data/.railway-admin-bootstrapped` exists (later boots);
2. runs `./main --create-user` with `DATABASUS_ADMIN_EMAIL` / `DATABASUS_ADMIN_PASSWORD` (upstream validates the address and the 8-character minimum);
3. with `DATABASUS_DISABLE_SIGNUP=true`, sets `users_settings.is_allow_external_registrations = false` in one transaction (the same flag as Databasus settings, Allow external registrations);
4. writes the marker, then starts Databasus as a child process, forwards SIGTERM/SIGINT to it, and after it exits stops the embedded PostgreSQL with `pg_ctl -m fast` (upstream lets the container runtime SIGKILL it, which forces crash recovery on every redeploy and can reach `start.sh`'s `pg_resetwal` fallback).

It fails closed: a missing email or password, a failed `--create-user`, a failed sign-up update, or an upstream `start.sh` that no longer ends in `exec gosu databasus ./main` stops the container before the HTTP server starts. The marker means the variables are read once: later password changes happen in the UI, and changing the admin's email in the UI cannot make a later boot create a second administrator.

Readable source (`template.json` holds the same statements joined into one line inside `/bin/bash -c '...'`, which the tests below compare byte for byte):

```bash
set -eu;
if ! grep -Eq "^[[:space:]]*exec gosu databasus [.]/main[[:space:]]*$" /app/start.sh; then
  echo "railway: unexpected /app/start.sh (no exec gosu databasus ./main), refusing to start without the admin bootstrap" >&2;
  exit 1;
fi;
RAILWAY_REAL_GOSU="$(command -v gosu)";
export RAILWAY_REAL_GOSU;
railway_pg_stop() {
  "$RAILWAY_REAL_GOSU" databasus /usr/lib/postgresql/17/bin/pg_ctl -D /databasus-data/pgdata -m fast -w -t 30 stop || true;
};
railway_bootstrap() {
  local marker=/databasus-data/.railway-admin-bootstrapped;
  if [ -f "$marker" ]; then
    echo "railway: admin already provisioned on this volume";
    return 0;
  fi;
  local email="${DATABASUS_ADMIN_EMAIL:-}";
  local password="${DATABASUS_ADMIN_PASSWORD:-}";
  if [ -z "$email" ] || [ -z "$password" ]; then
    echo "railway: DATABASUS_ADMIN_EMAIL and DATABASUS_ADMIN_PASSWORD are required, refusing to start without an administrator" >&2;
    return 1;
  fi;
  echo "railway: creating admin $email before the web server starts";
  if ! "$RAILWAY_REAL_GOSU" databasus ./main --create-user --email="$email" --password="$password" --role=admin; then
    echo "railway: admin creation failed, see the error above" >&2;
    return 1;
  fi;
  if [ "${DATABASUS_DISABLE_SIGNUP:-true}" = "true" ]; then
    if ! /usr/lib/postgresql/17/bin/psql -X -q -1 -v ON_ERROR_STOP=1 -d "$DATABASE_DSN" -c "INSERT INTO users_settings (is_allow_external_registrations) SELECT false WHERE NOT EXISTS (SELECT 1 FROM users_settings)" -c "UPDATE users_settings SET is_allow_external_registrations = false"; then
      echo "railway: could not turn off sign-up, set DATABASUS_DISABLE_SIGNUP=false to skip this step" >&2;
      return 1;
    fi;
    echo "railway: open sign-up turned off (an admin can turn it back on in Databasus settings)";
  fi;
  "$RAILWAY_REAL_GOSU" databasus touch "$marker";
};
railway_on_term() {
  if [ -n "${RAILWAY_MAIN_PID:-}" ]; then
    kill -TERM "$RAILWAY_MAIN_PID" 2>/dev/null || true;
  fi;
};
railway_run_main() {
  trap railway_on_term TERM INT;
  "$RAILWAY_REAL_GOSU" databasus ./main &
  RAILWAY_MAIN_PID=$!;
  local status=0;
  wait "$RAILWAY_MAIN_PID" || status=$?;
  while kill -0 "$RAILWAY_MAIN_PID" 2>/dev/null; do
    status=0;
    wait "$RAILWAY_MAIN_PID" || status=$?;
  done;
  echo "railway: Databasus exited with status $status, stopping embedded PostgreSQL";
  railway_pg_stop;
  exit "$status";
};
railway_shim() {
  if [ "$#" -eq 2 ] && [ "$1" = databasus ] && [ "$2" = ./main ]; then
    if ! railway_bootstrap; then
      railway_pg_stop;
      exit 1;
    fi;
    railway_run_main;
  fi;
  exec "$RAILWAY_REAL_GOSU" "$@";
};
mkdir -p /tmp/railway-gosu-shim;
{
  echo "#!/bin/bash";
  declare -f railway_pg_stop railway_bootstrap railway_on_term railway_run_main railway_shim;
  echo "railway_shim \"\$@\"";
} > /tmp/railway-gosu-shim/gosu;
chmod 0755 /tmp/railway-gosu-shim/gosu;
export PATH="/tmp/railway-gosu-shim:$PATH";
exec /app/start.sh
```

Paths it relies on, all from the v3.61.0 image: `/usr/sbin/gosu` (Debian bookworm `gosu` package, resolved with `command -v`), `/usr/lib/postgresql/17/bin/{psql,pg_ctl}` (the same directory `start.sh` uses), `DATABASE_DSN` (exported by `start.sh` before its final `exec`).

## Required input

`DATABASUS_ADMIN_EMAIL` only: the email you sign in with. Everything else has a working default. Without it the service refuses to start (by design, see above).

## First login

1. Deploy, wait for the healthcheck (first boot initializes PostgreSQL, runs migrations and creates the admin; allow 1 to 3 minutes).
2. Open the Databasus URL. The page opens on the Sign up form (upstream's default view); switch to **Sign in** and use `DATABASUS_ADMIN_EMAIL` and `DATABASUS_ADMIN_PASSWORD` (Variables tab of the Databasus service). Sign-up attempts by strangers are refused because open registration is off. Change the password under your profile.
3. **Storages**, add **S3**: Endpoint = `S3_ENDPOINT`, Region = `S3_REGION` (`auto`), Bucket = `S3_BUCKET`, Access key = `S3_ACCESS_KEY_ID`, Secret key = `S3_SECRET_ACCESS_KEY`, tick **Use virtual-hosted style** (new Railway Buckets are virtual-hosted; Databasus' minio-go client otherwise sends path-style requests to non-AWS endpoints; leave it off only if the Bucket's Credentials tab says path-style). Optional prefix, e.g. `databasus/`. Use the storage test button.
   Fallback: **Local** storage writes to `/databasus-data/backups` on the service volume (5 GB by default on Hobby, shared with the metadata database).
4. **Notifiers**: add Slack, Telegram, Discord, Teams, Mattermost or a webhook (all HTTPS, any plan). The Email notifier speaks SMTP, which Railway allows on Pro only.
5. **Add database** (see below), choose schedule, storage, retention (time, count or GFS) and notifiers. Keep backup encryption on (the UI default) and save `secret.key` (see Keep the key).
6. Run a first backup manually and check the object in the Bucket's Files tab.

Lost the admin password: `railway ssh --service Databasus`, then `cd /app && ./main --new-password="{new-password}" --email="{admin-email}"` (upstream's reset command). Changing `DATABASUS_ADMIN_PASSWORD` after the first boot has no effect. More users: add them to a workspace by email (they then sign up with that address, which works while open registration is off), or create them from the console with `./main --create-user --email=... --password=... --role=member`.

### Adding a Railway Postgres as a source

- Deploy this template **into the project that holds the database** (project canvas, Create, Template), so Databasus reaches it over the private network. Private traffic is free and not exposed.
- In **Add database**, PostgreSQL, paste the source's `DATABASE_URL` (Variables tab of the Postgres service: `postgresql://postgres:{password}@postgres.railway.internal:5432/railway`), or enter the host by hand (the Postgres service's `RAILWAY_PRIVATE_DOMAIN`, usually `postgres.railway.internal`), port 5432, user, password, database `railway`. SSL mode `disable` is fine on the private network (Railway's private network is encrypted); `require` also works with the image's self-signed certificate.
- Pick PostgreSQL version 18 for current Railway Postgres (`postgres-ssl:18`); Databasus ships client tools for 14 to 18.
- Accept **Create read-only user**: Databasus warns when given a superuser and creates a role that can only read. It then stores only the read-only credentials.
- Database in another Railway project or outside Railway: use its public URL (Railway: `DATABASE_PUBLIC_URL` via the TCP proxy) with SSL mode `require`. Every backup then leaves the source over the public network and is billed as the source's egress.
- MySQL, MariaDB and MongoDB work the same way (`mysql.railway.internal:3306`, `mongodb.railway.internal:27017`), logical backups only.

## Restore

- Logical backups (the normal case): open the database, pick a backup, **Restore from this backup**, and enter the connection of the target database. Create an empty target first (for example a second Railway Postgres or a new database); Databasus runs `pg_restore --clean --if-exists` (or the MySQL/MariaDB/MongoDB equivalent) and the target's current data is replaced. Restoring into the source database itself is possible but overwrites it.
- Default single-CPU restores stream from the Bucket into `pg_restore`. Parallel restores or restores with exclusions download the backup to `/databasus-data/temp` first, so they need free volume space equal to the backup size.
- **Download** gives a decrypted file for manual `pg_restore` / `mysql` / `mongorestore`. Encrypted objects in the Bucket cannot be used directly without Databasus or its `secret.key`.
- Upstream also documents recovery without Databasus (`databasus.com/how-to-recover-without-databasus`), which needs `secret.key`.

### Keep the key

`/databasus-data/secret.key` encrypts stored credentials and every encrypted backup. If the volume is lost, the backups in the Bucket cannot be decrypted without it. After the first deploy run `railway ssh --service Databasus -- cat /databasus-data/secret.key` (or use `railway volume browse`) and store the key in a password manager. Also enable Railway volume backups on the Databasus service (Backups tab) to keep the metadata database (schedules, backup index) recoverable.

## PITR (honest status)

Databasus v3.61.0 offers point-in-time recovery only through **physical** PostgreSQL backups: PostgreSQL 17 or newer, `pg_basebackup` (full and `--incremental` with `summarize_wal = on`) plus continuous WAL streaming with `pg_receivewal`, all over the **replication protocol** with a role that has `REPLICATION` (FAQ and `adr/` in the upstream repo; the old agent mode was removed after v3.42.0).

Against Railway-managed Postgres this does not work out of the box:

- Railway's `postgres-ssl` image allows remote clients with `host all all all scram-sha-256` (official entrypoint plus `wrapper.sh`). In `pg_hba.conf`, `all` does not match replication connections, so `pg_basebackup` from another service is refused until someone adds a `host replication ...` line on the database volume. `summarize_wal` is off by default.
- Even with those changes, a physical restore needs an empty data directory on a host you control and Databasus' recovery script; it cannot be poured back into a Railway-managed Postgres service in place.

So this template does not claim PITR for Railway Postgres. Use Databasus for scheduled logical backups with retention, encryption, alerts and restore into any database, and use **Railway's own Point-in-Time Recovery** (Postgres service, Backups tab: pgBackRest WAL archiving to a bucket, restore to a new service) when you need second-level recovery of a Railway Postgres. Physical/PITR mode in Databasus remains available for self-managed PostgreSQL 17+ servers that accept replication connections; that path was not tested here.

## Notifications and Railway plan limits

- HTTPS notifiers (Slack, Telegram, Discord, Microsoft Teams, Mattermost, webhook) work on every plan.
- Email notifier and instance mail (`SMTP_*`: invitations, password reset codes, sign-in codes) use SMTP. Railway allows outbound SMTP only on Pro (Rule 8.11). Databasus has no HTTP email API integration; on Trial/Hobby use an HTTPS notifier, or a webhook into a mail service of your own.
- Two-factor sign-in needs working instance mail; do not enable it without SMTP (recovery: `railway ssh --service Databasus`, then `cd /app && ./main --disable-2fa`, upstream's console command).
- Restore verification (automatic test restores) needs a separate verification agent on a host with Docker; Railway services cannot run Docker, so it is not part of this template.

## Improvements over market templates

1. **No open admin claim**: the administrator is created from Railway variables before the web server listens, and open self-registration is turned off (market: first visitor becomes admin, sign-up open).
2. **Fails closed**: no admin email, a rejected account or an unexpected upstream `start.sh` stops the deploy instead of serving an unclaimed instance.
3. **Clean shutdown of the embedded PostgreSQL** on every redeploy (`pg_ctl -m fast` after Databasus exits, 60 s draining), instead of SIGKILL and crash recovery.
4. **Healthcheck and `PORT`** (`/api/v1/system/version`, 600 s for first-boot migrations).
5. **Bucket wired in**: its credentials appear on the service, with the exact form mapping and the virtual-hosted style setting Railway Buckets need.
6. **Railway-specific docs**: private-network source setup, read-only user, restore flow, `secret.key` custody, honest PITR status with the Railway alternative, plan limits for alerts.
7. Compared with Railway's Postgres Backup cron: current `pg_dump` versions up to PostgreSQL 18, MySQL/MariaDB/MongoDB, retention (time, count, GFS), encryption, alerts, a UI and restore.
8. stdout-only logging (no rotating log file on the volume), all variables documented.

## Resource estimate

Railway prices: RAM $10/GB-month, vCPU $20/vCPU-month, volume $0.15/GB-month, Bucket $0.015/GB-month, egress $0.05/GB (uploads to a Bucket count as service egress; private network traffic is free). No local measurement (no Docker this round); figures come from upstream's sizing (minimum 500 MB RAM, Helm requests 1 GiB / 0.5 CPU), the embedded PostgreSQL settings (`shared_buffers=256MB`, mostly untouched at idle) and the streaming design (one 16 MiB part in flight).

| Item | Idle | Typical | Notes |
|---|---|---|---|
| Databasus RAM | 0.2 GB | 0.35 GB average | Go backend + embedded PostgreSQL; `pg_dump` and zstd add a few hundred MB while a job runs |
| Databasus CPU | 0.01 vCPU | 0.05 vCPU average | bursts to ~1 vCPU during compression |
| Volume | 0.3 GB | 0.5 GB | metadata DB; more if you use local storage |
| Bucket | 0 | 15 GB | e.g. a week of daily 2 GB compressed dumps |
| Egress | 0 | 30 GB | e.g. 1 GB of compressed backups uploaded per day |

Estimated monthly cost: **idle ≈ $2.25** (0.2 GB × $10 + 0.01 × $20 + 0.3 GB × $0.15), **typical ≈ $6.30** (0.35 GB × $10 + 0.05 × $20 + 0.5 GB × $0.15 + 15 GB × $0.015 + 30 GB × $0.05). Large databases scale the Bucket and egress lines linearly; the source database's own usage is not included.

Minimum plan: **Hobby** recommended. Trial (1 GB RAM, 0.5 GB volume) can boot the template and back up small databases to the Bucket, but leaves no room for local storage or file-based restores. Hobby gives 5 GB of volume per service (resizable).

## Upgrades

Change the image tag (for example `v3.62.0`), read the release notes, make sure `secret.key` and a volume backup exist, redeploy. Databasus migrates its metadata database on start. The start command checks that the new `/app/start.sh` still ends with `exec gosu databasus ./main`; if upstream changes that line the deploy stops with `railway: unexpected /app/start.sh` instead of starting without the admin bootstrap. Then compare the new `docker/start.sh` and adjust the grep and the shim condition.

## Verification checklist

Static checks done this round (details in `_reports/round4/61-databasus.md`): registry checks of both tags, image config, `bash -n` on the readable and the embedded command, a stub-based simulation of the start command (33 checks), `template.json` parsing and conversion with `_tools/to_serialized.py`, upstream source reading. No container was run.

On first Railway deploy:

- [ ] Deploy log shows `railway: creating admin {email} before the web server starts`, `Created account ... It is the administrator of this instance.`, `railway: open sign-up turned off`, then the Databasus start-up; healthcheck `/api/v1/system/version` passes and returns `{"version":"3.61.0"}`.
- [ ] Deploy without `DATABASUS_ADMIN_EMAIL` fails with `railway: DATABASUS_ADMIN_EMAIL and DATABASUS_ADMIN_PASSWORD are required` and never serves the UI.
- [ ] Sign-in with the admin email and generated password works; the sign-up form is closed (Databasus settings shows Allow external registrations off).
- [ ] `/api/v1/system/health` returns 200.
- [ ] S3 storage with the Bucket values and Use virtual-hosted style passes the storage test; a manual backup of a Railway Postgres 18 over `postgres.railway.internal` lands in the Bucket; a Slack/Telegram/webhook notifier fires.
- [ ] Restore of that backup into a second, empty Postgres succeeds.
- [ ] Redeploy: log shows `railway: admin already provisioned on this volume`, `railway: Databasus exited with status 0, stopping embedded PostgreSQL`, and on the next start no `pg_resetwal`/crash-recovery messages; schedules, storages and backup history are still there.

## Open questions

- Railway Bucket with `Use virtual-hosted style` and region `auto` through minio-go v7.2.1: expected to work (Railway documents virtual-hosted URLs and region `auto`), not tested.
- Signal path on Railway: the shim assumes `podman-init` forwards SIGTERM to its direct child (the shim). If it signals the whole process group instead, PostgreSQL gets a smart shutdown request directly and `pg_ctl` finds it already stopping; both paths end cleanly in the stub test, not tested on Railway.
- The simulation ran with macOS bash 3.2; the image has Debian bash 5.2. The constructs used (`declare -f`, `wait` with traps, `local`) behave the same, but this is unverified in the real image.
- The shim lives in `/tmp/railway-gosu-shim` and must be executable there (normal for Railway containers; a `noexec` `/tmp` would stop the first `gosu` call in `start.sh`, failing closed). The start command is 2,818 characters; Railway has accepted 537-character start commands in earlier tests, and no limit is documented.
- `--create-user` runs migrations and the client-tool check once more before the web process does, which lengthens first boot by some seconds; the 600 s healthcheck window should cover it.
- Databasus physical/PITR backups against a Railway Postgres after manual `pg_hba.conf` and `summarize_wal` changes: not attempted.
