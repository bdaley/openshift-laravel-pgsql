# OpenShift S2I Prep Guide

This repository is prepared for Source-to-Image (S2I) usage on OpenShift with these constraints:

- Scope: repository-only S2I enablement
- Database: keep SQLite
- Migrations: run from a separate OpenShift Job, not app startup

## What Was Added

- `.s2i/bin/assemble`
	- Installs production PHP dependencies
	- Installs Node dependencies
	- Builds Vite assets
	- Clears Laravel caches
- `.s2i/bin/run`
	- Starts the app runtime in foreground
	- Does not run database migrations
- `.s2i/bin/save-artifacts`
	- Saves `vendor` and `node_modules` for incremental builds

## Runtime Assumptions

- App root: `/var/www/html`
- Public web root: `/var/www/html/public`
- Health endpoint: `/up`
- Runtime user: `www-data` (non-root)

## Required Environment Variables

At minimum:

- `APP_ENV=production`
- `APP_DEBUG=false`
- `APP_KEY=<generated-secret>`
- `APP_URL=https://<your-route>`
- `DB_CONNECTION=sqlite`
- `DB_DATABASE=/var/www/html/database/database.sqlite`

Recommended for containers:

- `LOG_CHANNEL=stderr`

Generate a key locally:

```bash
php artisan key:generate --show
```

## Persistent Volume Expectations

Because SQLite is retained, both paths must be writable and persistent:

- `/var/www/html/storage`
- `/var/www/html/database`

If these are not persistent, sessions, cache, logs, and database data can be lost on pod restart.

## Migration Job Strategy

Do not run migrations in app startup.

Use a separate OpenShift Job (or equivalent rollout step) with:

```bash
php artisan migrate --force --no-interaction
```

Expected order:

1. Build image via S2I.
2. Deploy app image.
3. Run migration Job.
4. Confirm readiness and route traffic.

## Local Verification Commands

From repository root:

```bash
chmod +x .s2i/bin/assemble .s2i/bin/run .s2i/bin/save-artifacts
```

```bash
php artisan test --compact
```

## Troubleshooting

- Permission denied on `storage` or `database`:
	- Ensure mounted volumes are writable by the runtime UID/GID used by the container.
- Missing Vite manifest errors:
	- Ensure `npm run build` completed in S2I assemble.
- App starts but fails readiness:
	- Verify `APP_KEY`, `APP_URL`, and SQLite path env values.
- Migration failures:
	- Run migrations via job and verify target database path is mounted and writable.
