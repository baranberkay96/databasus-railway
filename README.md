# Deploy and Host Databasus with Railway

[![Deploy on Railway](https://railway.com/button.svg)](https://railway.com/deploy/databasus-v3-postgres-backups-and-restor?referralCode=20H68q)

Databasus is an open source (Apache-2.0) tool that backs up PostgreSQL, MySQL, MariaDB and MongoDB on a schedule, keeps the files in S3 or other storage, alerts you on success or failure and restores from a web dashboard. This community template runs Databasus 3.61.0 with a Railway Bucket for backup storage and a locked-down admin account.

## About Hosting Databasus

Databasus is a single Go service with a built-in web UI. It keeps its configuration, backup history and encrypted credentials in an embedded PostgreSQL 17 on its own volume, runs `pg_dump`, `mysqldump` or `mongodump` against your databases, compresses and encrypts the stream and uploads it to the storage you choose. Hosting it well means a persistent volume, a safe first login, reachable source databases and somewhere durable for the files. This template pins the official v3.61.0 image, creates your administrator from Railway variables before the dashboard goes online, turns off open sign-up, adds a Railway Bucket for off-volume backups, sets a healthcheck and stops the embedded database cleanly on every redeploy.

## Common Use Cases

- Nightly or hourly backups of the Postgres, MySQL or MongoDB services in a Railway project, kept in a Railway Bucket with time, count or GFS retention
- One dashboard for backups of databases across several projects or providers (Railway, RDS, Supabase, Neon, self-hosted)
- Slack, Telegram, Discord, Teams, Mattermost or webhook alerts when a backup fails or succeeds
- Restoring a backup into a fresh database to recover data, debug an incident or seed a staging environment
- Encrypted off-site copies, with read-only database users so the backup tool cannot change your data

## Dependencies for Databasus Hosting

- A persistent volume for the embedded metadata database and the encryption key (included, `/databasus-data`)
- S3-compatible object storage for backup files (included: a Railway Bucket); local volume storage works as a fallback
- The databases you want to back up, reachable from Databasus (same Railway project over the private network, or a public connection URL)
- Optional: a Slack, Telegram, Discord, Teams or Mattermost webhook for alerts; an SMTP server for email (Railway Pro plan only)

### Deployment Dependencies

- Databasus repository: https://github.com/databasus/databasus
- Databasus documentation: https://databasus.com/installation
- Advanced configuration (environment variables): https://databasus.com/advanced-config
- Account and password commands: https://databasus.com/password
- Security model: https://databasus.com/security
- Railway Buckets: https://docs.railway.com/storage-buckets
- Railway Point-in-Time Recovery: https://docs.railway.com/volumes/point-in-time-recovery

### Implementation Details

| Service | Image | Purpose |
|---|---|---|
| Databasus | `databasus/databasus:v3.61.0` | Web UI and API on port 4005 (public, sign-in required), backup scheduler, embedded PostgreSQL 17 for metadata on the `/databasus-data` volume; healthcheck `/api/v1/system/version` |
| Bucket | Railway Bucket | S3-compatible storage for backup files |

**First login**

1. Enter your email as `DATABASUS_ADMIN_EMAIL` when you deploy (the only required value). The administrator account is created with it on first boot, before the web server starts, and open self-registration is switched off. A generated password is in `DATABASUS_ADMIN_PASSWORD` on the Databasus service.
2. Open the Databasus URL. It shows the Sign up form first; switch to **Sign in** and use your email and the generated password. Change the password under your profile.
3. **Storages**, add **S3**, and copy the values from the Databasus service variables: `S3_ENDPOINT`, `S3_REGION` (auto), `S3_BUCKET`, `S3_ACCESS_KEY_ID`, `S3_SECRET_ACCESS_KEY`. Tick **Use virtual-hosted style** (new Railway Buckets use it). Local storage on the volume is the fallback.
4. **Notifiers**: add Slack, Telegram, Discord, Teams, Mattermost or a webhook.
5. **Add database**, set the schedule, storage, retention and notifiers, keep encryption on, and run a first backup.
6. Save the encryption key: `railway ssh --service Databasus -- cat /databasus-data/secret.key`. Encrypted backups cannot be read without it if the volume is ever lost. Enabling Railway volume backups on the Databasus service also protects schedules and backup history.

**Adding a Railway Postgres**

Deploy the template into the same project as the database so the two talk over Railway's private network. In **Add database**, PostgreSQL, paste the source's private `DATABASE_URL` from its Variables tab (`postgresql://postgres:{password}@postgres.railway.internal:5432/railway`), or enter the host `postgres.railway.internal` (the value of `RAILWAY_PRIVATE_DOMAIN` on the Postgres service), port 5432, user, password and database. SSL can stay off on the private network, which is already encrypted. Choose PostgreSQL 18 for current Railway Postgres. When Databasus offers to **create a read-only user**, accept: it then keeps only credentials that cannot modify data. For a database in another project or outside Railway, use its public URL (`DATABASE_PUBLIC_URL` on Railway) with SSL mode `require`; that traffic is billed as egress. MySQL, MariaDB and MongoDB are added the same way.

**Restoring**

Open a backup and choose **Restore from this backup**, then enter the connection of the target database. Create an empty target first, such as a second Railway Postgres: the restore replaces whatever the target holds. You can also download a decrypted backup file and restore it yourself with `pg_restore`, `mysql` or `mongorestore`.

**Point-in-time recovery**

Databasus 3.61 does point-in-time recovery only with physical PostgreSQL 17+ backups over the replication protocol, restored into an empty data directory on a host you control. Railway's managed Postgres does not accept replication connections from other services by default and cannot be restored in place that way, so this template does not offer PITR for Railway Postgres. Use Databasus for scheduled logical backups and restores, and Railway's built-in Point-in-Time Recovery (Backups tab of the Postgres service) when you need to rewind a Railway Postgres to a specific second.

**Alerts and email**

Slack, Telegram, Discord, Teams, Mattermost and webhook notifiers work on every Railway plan. The email notifier and the instance mail settings (`SMTP_HOST`, `SMTP_PORT`, `SMTP_USER`, `SMTP_PASSWORD`, `SMTP_FROM`, `SMTP_SECURITY`) use SMTP, which Railway allows only on the Pro plan. Databasus has no HTTP email API integration, so on Trial or Hobby use one of the other notifiers. Automatic restore verification needs a separate agent on a host with Docker and is not part of this template.

**Scaling**: Databasus runs as one instance with its own volume; do not add replicas. For large databases give the service more CPU and RAM (compression is CPU-bound) and raise the CPU count per database in Databasus for parallel dumps. If you keep backups on the volume instead of the Bucket, resize the volume.

**Pinning and upgrades**: the image is pinned to `v3.61.0`. To upgrade, save `secret.key`, take a volume backup, change the tag and redeploy; Databasus migrates its metadata database on start. The start command checks that the image's startup script still has the shape it expects and stops the deploy with a clear message if it does not, instead of starting without the admin bootstrap.

**Account recovery**: `DATABASUS_ADMIN_PASSWORD` is read on the first boot only. To reset a password later, open a shell with `railway ssh --service Databasus` and run `cd /app && ./main --new-password="{new-password}" --email="{your-email}"`.

**Resources**: about 0.2 GB of RAM at idle, roughly $2 to $3 per month; with daily backups of a few GB expect about $6 to $7 per month including Bucket storage and upload egress. Hobby or higher is recommended.

### Why Deploy Databasus on Railway?

Railway puts Databasus next to the databases it protects: backups run over the private network, files land in a Railway Bucket in the same project, and the dashboard gets HTTPS without extra setup. You keep control of your data and keys, and the defaults are locked down from the first deploy.
