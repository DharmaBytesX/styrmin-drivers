# Nextcloud driver for Styrmin

Deploys Nextcloud (Apache image, `docker.io/library/nextcloud:<version>-apache`) with a PostgreSQL database, using the official chart `https://nextcloud.github.io/helm/nextcloud` 9.4.1 (application 35.0.2). Supported application versions: `>=35.0,<35.1`.

| Component | Workload | Backup |
|---|---|---|
| `app` | Deployment/styrmin-nextcloud | disk snapshot of volume `nextcloud-main` (`/var/www/html`) |
| `postgresql` | StatefulSet/styrmin-postgresql (Bitnami legacy image 17.5.0) | `pg_dump` into `/backup` |

## Load and deploy

    styrminctl drivers load-version <git-url> <branch> --driver-directory <path/in/repo>
    styrminctl deployments create nextcloud 35.0.2 <environment> --cluster <cluster>

## Credentials

- Nextcloud administrator: user `admin`, password `styrmin-nextcloud-admin` (`nextcloud.password` in `values.j2.yml`).
- PostgreSQL: user `nextcloud`, database `nextcloud`, password `styrmin-nextcloud-db` (`postgresql.global.postgresql.auth.password`).
- These are fixed, public values. Change them in `values.j2.yml` before any real use. Changing the administrator password later does not change an existing installation; use the Nextcloud user settings.

## Parameters

- `storage_size` (default `20Gi`): size of the volume for `/var/www/html`.

## Decisions

- PostgreSQL rather than the chart's default SQLite (`internalDatabase`) or MariaDB: SQLite is not suitable for several users, and the skill's backup strategy `postgres` is supported.
- Redis is left off. The chart builds the Redis host from the release name (`<release>-redis-master`), but the driver must use `fullnameOverride` on subcharts, so the application would not find it. Without Redis, Nextcloud uses the database for file locking and local caching, which is slower under heavy use. Not verified: whether the release name is known when the driver is deployed, which would allow Redis again.
- Collabora and MariaDB subcharts are off: not needed for a first driver.
- The chart supports one Ingress host. A second Ingress (`styrmin-nextcloud-readable`) in `extraManifests` serves `human_readable_fqdn`; both hosts are in `trustedDomains`.
- `phpClientHttpsFix` is enabled when the external URL scheme is `https`, so Nextcloud builds https links behind the Styrmin ingress.
- Image tag is `{{ app.version }}-apache` because the chart's default tag has that suffix.
- Egress NetworkPolicy allows DNS, traffic inside the deployment and public internet (excluding private ranges), because Nextcloud uses the app store, update checks and federation.
- The `app` volume backup uses a disk snapshot. A consistent restore of files and database together was not considered beyond taking both backups.

## Checked

`check_driver.py --check-images`: 0 FAIL, 0 WARN. Both images exist for linux/amd64 and linux/arm64.

## Not verified

- Not deployed on a cluster. Pods being Ready within 270 s, the first-run installation, login, ingress behaviour with two hosts and backup and restore were not tested.
- Upgrading to another chart or application version was not tested.
