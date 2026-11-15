# Frappe ERP on Coolify (ERPNext + HR + CRM + Helpdesk + Raven + Insights)

This repo runs one Frappe stack on Coolify with these apps:

- **ERPNext**: accounting, invoicing, stock and buying/selling
- **Frappe HR** (`hrms`): employees, leave, attendance, payroll
- **Frappe CRM**: leads and deals, served at `/crm`
- **Helpdesk**: support tickets, served at `/helpdesk` (needs `telephony`, which is listed before it)
- **Raven**: team chat, served at `/raven`
- **Insights**: reports and dashboards, served at `/insights` (`version-3` branch; its `main` branch is the old 2.x line)
- **Offsite Backups** (`offsite_backups`): daily backups to Cloudflare R2 (see step 9)
- **Tuteome CRM** (`tuteome_crm`, private repo `tuteome/tuteome_crm`): Tuteome's sales and marketing layer on Frappe CRM.
  It isn't in `apps.json`; the workflow adds it from a secret (see "Private apps" in step 11)

All of them are installed on one site, so they share users and data. For example, a CRM deal can become an ERPNext customer and quotation.

## Why the image is self-built

The official `ghcr.io/frappe/crm` images have **not contained the crm app since v1.67.1**
([frappe/crm#2538](https://github.com/frappe/crm/issues/2538)).
`stable`, `v16.0.3` and `v1.92.0` all contain only Frappe, and `v1.67.0` is the last good one. CRM's build workflow
still passes `APPS_JSON_BASE64`, but frappe_docker now reads apps from an `apps_json` build secret.

`.github/workflows/build-image.yml` uses frappe_docker's documented layered build to produce
`ghcr.io/tuteome/frappe-erp` with frappe, erpnext, hrms and crm. The workflow fails if any of those apps
is missing from the image.

## Files

- `coolify-compose.yml`: the stack to paste into Coolify.
- `easy-install-exact.yml`: what `easy-install.py deploy` would generate. It's for reference only and uses the broken image.
- `apps.json`: the public apps baked into the image. Edit this file to add an app. Private apps are added by the workflow (step 11).
- `.github/workflows/build-image.yml`: the image build.

`coolify-compose.yml` is the easy-install stack with these changes:
- The script's Traefik and published ports are removed, because Coolify's proxy routes to `frontend:8080`.
- The DB root password is Coolify's generated `${SERVICE_PASSWORD_MARIADB}`.
- Config values are written out literally, because Coolify can't parse `$$VAR` escapes.
- `FRAPPE_SITE_NAME_HEADER` is left to nginx's default, `$host`.
- MariaDB stays on **11.8**, the version Frappe tests. Do not move to MariaDB 12.x: it fails to create the site, because
  HRMS's `ADD INDEX ... (employee, from_date, to_date)` is a syntax error there (`to_date` clashes with the newer
  `TO_DATE` function, and Frappe does not quote index columns).
- Redis is 8.10 instead of the script's 8.6.
- `backend` sets `FORWARDED_ALLOW_IPS: "*"`. Gunicorn only trusts `X-Forwarded-Proto` from `127.0.0.1` by default,
  but nginx runs in another container. Without this setting Frappe builds `http://` URLs, and the login page then
  drops OAuth redirects, which breaks sign-in to the Raven mobile app.

---

## Step-by-step

### 1. Build the image (about 30–45 min)

1. Open **Actions → Build Frappe ERP image → Run workflow**. Keep `frappe_branch` as `version-16` and run it.
2. Wait for a green run. The **Verify all apps** step prints each app's version. The run summary shows the
   exact tag, for example `ghcr.io/tuteome/frappe-erp:version-16-3`.

### 2. Let Coolify pull the image

The image contains the private `tuteome_crm` code, so the package **must stay private**
(github.com → your profile → **Packages → frappe-erp → Package settings**: visibility Private).

On the Coolify server, log in once so it can pull:

1. Create a GitHub token (classic) with only `read:packages`.
2. On the server, run `docker login ghcr.io -u tuteome` and paste the token as the password.

### 3. DNS

Create an A record, for example `erp.tuteome.com`, pointing to the Coolify server.
Use a general name, because this site holds the whole ERP and not just CRM. The site name can't easily be changed later.

### 4. Clean up the earlier failed attempt

In the existing Coolify resource, stop it, delete it, and also delete its volumes (`sites`, `db-data`,
`redis-queue-data`). The failed `new-site` left a half-installed Frappe 15 site behind, and nothing in it is worth keeping.

### 5. Create the Coolify resource

1. Go to **+ New → Docker Compose Empty** and paste in `coolify-compose.yml`.
2. Optionally change `image:` from `version-16` to the pinned `version-16-<run>` tag from step 1. A pinned tag means
   redeploys never pull a new version by surprise. If you pin it, replace all 7 occurrences.
3. In the **Environment Variables** tab, check that `SERVICE_PASSWORD_MARIADB` was generated. Copy it, because you need it in step 7.
4. Set the domain on the `frontend` service only: `https://erp.tuteome.com:8080`.
   The `:8080` is the container port. Coolify still serves the site on 443.
5. Leave every other service without a domain.

### 6. Deploy

Click **Deploy**. Then check the following:
- `configurator` runs once and exits with code 0. That's expected.
- `db` reports healthy.
- `backend`, `frontend`, `websocket`, `queue-short`, `queue-long`, `scheduler` and `cron` are running.

At this point, opening the domain shows a "site not found" error. That's expected until step 7.

### 7. Create the site (once)

Open Coolify's **Terminal**, select the `backend` container, and run:

```
bench new-site erp.tuteome.com \
  --mariadb-user-host-login-scope='%' \
  --db-root-password='<SERVICE_PASSWORD_MARIADB>' \
  --admin-password='<choose a strong one>' \
  --install-app erpnext --install-app hrms --install-app crm \
  --install-app telephony --install-app helpdesk --install-app raven --install-app insights \
  --install-app offsite_backups --install-app tuteome_crm
```

- The site name **must equal the domain**, because nginx picks the site from the Host header.
- Keep the app order: hrms depends on erpnext.
- If it prompts `Enter mysql super user [root]:`, press Enter.
- This takes a few minutes.

Then run:

```
bench --site erp.tuteome.com enable-scheduler
```

### 8. First login

- `https://erp.tuteome.com/app`: log in as `Administrator` with the admin password, then complete the ERPNext setup
  wizard (company, currency, fiscal year, chart of accounts).
- `https://erp.tuteome.com/crm`: CRM.
- HR is under `/app` (Employee, Leave, Attendance and Payroll in the sidebar).

Create real user accounts for people. Don't share the Administrator login.

### 9. Backups

The `cron` (ofelia) container runs `bench --site all backup` at 00:00, 06:00, 12:00 and 18:00 UTC. It writes
database dumps to `sites/erp.tuteome.com/private/backups`. Attachments, Raven's included, stay on the server in
`public/files` and `private/files`. So the `sites` volume holds everything a restore needs: the files,
`site_config.json` (with the `encryption_key`), and recent database dumps.

Off-server copies go to Cloudflare R2 in two independent ways. Use a **private** bucket for each, or one bucket with
separate paths, and an R2 API token with **Object Read & Write** on that bucket only. Both copies contain
`site_config.json`, which holds the DB password and the `encryption_key`, so the bucket must stay private.

**Primary: the `offsite_backups` app.** It produces Frappe's own backup format, so restoring is a single `bench restore`.
Frappe v16 moved S3, Google Drive and Dropbox backups out of core and into this app.

1. In `/app/s3-backup-settings`, fill in these fields:
   - **Endpoint URL**: `https://<account-id>.r2.cloudflarestorage.com`
   - **Access Key ID** and **Secret Access Key**: from the R2 token
   - **Bucket**: the bucket name. **Backup Files**: on. **Frequency**: Daily. **Notify Email**: your address.

   Tick **Enabled** and save. Saving checks that the bucket is reachable. **Take Backup Now** runs a test upload.
2. In R2, add a lifecycle rule that deletes objects after, for example, 30 days. Every upload is a full copy,
   and the app never deletes old copies.
3. To restore, run:
   `bench --site <site> restore <database.sql.gz> --with-public-files <files.tar> --with-private-files <private-files.tar>`

**Secondary: a Coolify storage backup of the `sites` volume.** It still works if Frappe itself is broken.

1. Add R2 in Coolify under **Storages** (S3, same endpoint).
2. Open the resource → **Backups → Add Backup → storage backup**. For the target, pick **one** `…: sites` entry
   (they're all the same volume), for example `Backend: sites`. Set the frequency to `30 0 * * *`, which runs
   30 minutes after the 00:00 UTC dump. Adjust it if Coolify isn't on UTC. Then set how many backups to keep.
3. **Don't** back up `db-data`. A copy of MariaDB's files taken while it's running can be inconsistent, and the
   dumps inside `sites` are the safe database backup. `redis-queue-data` doesn't need a backup.
4. To restore, extract the archive, run `bench --site <site> restore <private/backups/...-database.sql.gz>`,
   and copy `public/files` and `private/files` back into the site folder.

**Test a restore** from the primary backup once, into a scratch site.

### 10. Upgrades

1. Run the build workflow again. It builds the latest `version-16` commits and gives the image a new `version-16-<run>` tag.
2. Take a backup: `bench --site all backup --with-files`.
3. Update the image tag in Coolify and redeploy.
4. In the `backend` terminal, run `bench --site all migrate`.

To move to a new release line (for example `version-17`), check that erpnext, hrms and crm all have that branch first.

### 11. Adding another Frappe app

1. **Check compatibility.** The app must have a branch for your release line (`version-16`). Many Frappe apps,
   such as helpdesk, raven, lms and builder, use `main` instead (insights uses `version-3`).
   Check `[tool.bench.frappe-dependencies]` in the app's `pyproject.toml` for the Frappe range and any app it needs. Check the app's README for which Frappe versions it supports.
   If the app depends on another app, list the dependency first.
2. **Add one line to `apps.json`.** For example:
   ```json
   {"url": "https://github.com/frappe/helpdesk", "branch": "main"}
   ```
   Commit and push the change.
3. **Rebuild.** Run the workflow again. It fails if any app in `apps.json` is missing from the image.
   Note the new tag, for example `version-16-2`.
4. **Back up.** In the `backend` terminal, run `bench --site all backup --with-files`.
5. **Redeploy.** Update all 7 image tags in Coolify to the new tag and redeploy.
   The configurator refreshes `sites/apps.txt` automatically.
6. **Install the app on the site.** In the `backend` terminal, run:
   ```
   bench --site erp.tuteome.com install-app helpdesk
   bench --site erp.tuteome.com migrate
   ```

**Private apps:** this repo is public, so **never** put a token in `apps.json`. Private apps are added by the
workflow's **Add private apps** step instead:

- It merges `apps.json` with `https://x-access-token:<token>@github.com/tuteome/tuteome_crm` (branch `main`)
  into a temp file outside the checkout, and passes that file as the `apps_json` build secret. The file is never printed.
- The token is the repository secret **`TUTEOME_CRM_TOKEN`**: a fine-grained token, resource owner `tuteome`,
  only the repository `tuteome/tuteome_crm`, permission **Contents: Read-only**. Set it under
  **Settings → Secrets and variables → Actions**. Renew it before it expires; the build fails clearly when it's missing.
- The workflow checks the token isn't left anywhere in the image (frappe_docker also strips every `apps/*/.git`).
- Another private app (e.g. `tuteome_hr`) gets its own token and line in that step.

**Releasing a `tuteome_crm` change:** push to `tuteome_crm`'s `main` (CI green), run this workflow, then follow
step 10 (backup, new tag in Coolify, `bench --site all migrate`). The app's setup re-applies itself on migrate.

To remove an app, run `bench --site erp.tuteome.com uninstall-app <app>` first, then remove it from `apps.json` and rebuild.
The order matters: if you remove an app from the image while it's still installed on the site, the site breaks.

---

## Troubleshooting

- **`No module named 'crm'`** (or erpnext/hrms): you're running an image without that app. Check the image tag.
- **"Site not found"**: either the site name doesn't match the domain, or the site hasn't been created yet.
- **Access denied for root**: the database was initialised with a different password. MariaDB only reads the
  root password when it first creates `db-data`. Use the original password, or wipe `db-data` if there's no data yet.
- **Coolify shows "The key must start with a letter or underscore…"**: a `$$VAR` or `${VAR:-default}`
  expression got into the compose file. Write the value out literally instead.
