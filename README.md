# OpenShift S2I Prep Guide

This repository is prepared for Source-to-Image (S2I) usage on OpenShift with these constraints:

- Scope: repository-only S2I enablement
- Database: keep SQLite
- Migrations: run from a separate OpenShift Job, not app startup

## What Was Added

- `.s2i/bin/assemble`
	- Restores incremental build artifacts when present
	- Installs production PHP dependencies
	- Installs Node dependencies
	- Builds Vite assets
	- Ensures Laravel writable runtime paths exist
	- Clears Laravel caches
- `.s2i/bin/run`
	- Starts a builder-image-centric foreground runtime
	- Does not run database migrations
- `.s2i/bin/save-artifacts`
	- Saves `vendor` and `node_modules` for incremental builds

## Local vs OpenShift

- Local development keeps using serversideup via `Dockerfile` and `docker-compose.yaml`.
- OpenShift uses Source strategy builds and `.s2i/bin/*` scripts.
- OpenShift should not use the Dockerfile when BuildConfig strategy is `Source`.

## Builder Image Selection

Use the parameterized template at `openshift/buildconfig-s2i.yaml`.

Default builder:

- `BUILDER_KIND=DockerImage`
- `BUILDER_IMAGE=registry.access.redhat.com/ubi9/php-84`

Template apply example:

```bash
oc -n laravel-staging process -f openshift/buildconfig-s2i.yaml \
	-p APP_NAME=laravel-web \
	-p NAMESPACE=laravel-staging \
	-p GIT_URI=https://github.com/bdaley/openshift-laravel-pgsql.git \
	-p GIT_REF=main \
	-p BUILDER_KIND=DockerImage \
	-p BUILDER_IMAGE=registry.access.redhat.com/ubi9/php-84 \
	-p OUTPUT_IMAGESTREAM_TAG=laravel-web:latest | oc apply -f -
```

Builder override examples:

```bash
# Switch to an imagestream builder (if your cluster provides one)
oc -n laravel-staging process -f openshift/buildconfig-s2i.yaml \
	-p BUILDER_KIND=ImageStreamTag \
	-p BUILDER_IMAGE=php:8.4-ubi9 \
	-p BUILDER_NAMESPACE=openshift | oc apply -f -

# One-off patch to an existing BuildConfig
oc -n laravel-staging patch bc/laravel-web --type=merge -p '{"spec":{"strategy":{"type":"Source","sourceStrategy":{"from":{"kind":"DockerImage","name":"registry.access.redhat.com/ubi9/php-84"}}}}}'
```

Verify OpenShift ignores Dockerfile:

```bash
oc -n laravel-staging get bc/laravel-web -o jsonpath='{.spec.strategy.type}{"\n"}'
```

Expected output: `Source`

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

Minimal Job example:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: laravel-migrate
  namespace: laravel-staging
spec:
  ttlSecondsAfterFinished: 600
  backoffLimit: 1
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: migrate
          image: image-registry.openshift-image-registry.svc:5000/laravel-staging/laravel-web:latest
          command: ['php', 'artisan', 'migrate', '--force', '--no-interaction']
          envFrom:
            - configMapRef:
                name: laravel-web
            - secretRef:
                name: laravel-web
          volumeMounts:
            - name: storage
              mountPath: /var/www/html/storage
            - name: database
              mountPath: /var/www/html/database
      volumes:
        - name: storage
          persistentVolumeClaim:
            claimName: laravel-storage
        - name: database
          persistentVolumeClaim:
            claimName: laravel-database
```

Apply it with:

```bash
oc apply -f migration-job.yaml
oc -n laravel-staging logs -f job/laravel-migrate
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
