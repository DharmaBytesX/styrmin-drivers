# nextcloud driver for Styrmin

Deploys [Nextcloud](https://nextcloud.com/) 35.0.x from the official `nextcloud/nextcloud` Helm
chart: one Nextcloud web pod (Apache image) with its data on a 20Gi volume, a cron sidecar, and a
single-primary PostgreSQL database from the chart's Bitnami subchart with an 8Gi volume. The
external HTTP service is published through the chart's Ingress on the deployment's primary URL.
No Redis and no Collabora are deployed.

| Component | Workload | Backup |
|---|---|---|
| `app` | Deployment/styrmin-nextcloud (containers `nextcloud` and `nextcloud-cron`) | disk snapshot of volume `nextcloud-main` |
| `postgresql` | StatefulSet/styrmin-postgresql (container `postgresql`) | `pg_dump` into `/backup`, then a snapshot |

- Chart: `https://nextcloud.github.io/helm/nextcloud` 9.4.1 (application 35.0.2).
- Images: `docker.io/library/nextcloud:35.0.2-apache`,
  `docker.io/bitnamilegacy/postgresql:17.5.0-debian-12-r3`.
- Supported application versions: `>=35.0,<35.1` (patch releases of the 35.0 series; the image
  tag is `<app version>-apache`).

## Load and deploy

```bash
styrminctl drivers load-version <git-url> <branch> --driver-directory nextcloud-driver
styrminctl deployments create nextcloud 35.0.2 <environment> --cluster <cluster>
```

The Ingress is created by the chart; Styrmin sets its ingress class. The URL users open is
`<external_url_scheme>://<primary_fqdn>` (with the external port when it is not 80/443).

## Credentials

Every credential is fixed in `values.j2.yml`, not generated, so a restored or cloned deployment
keeps working with its data. Change them there **before** loading the driver, or change the
Nextcloud admin password in the Nextcloud UI after the first login.

| Credential | Default value |
|---|---|
| Nextcloud administrator | user `admin`, password `nextcloud-admin-password` |
| PostgreSQL application user | user `nextcloud`, password `nextcloud-db-password`, database `nextcloud` |
| PostgreSQL superuser | `postgres` / `nextcloud-postgres-password` |
| PostgreSQL replication user | `nextcloud-replication-password` |

PostgreSQL is a ClusterIP Service inside the deployment namespace; only Nextcloud reaches it.
SMTP is disabled (the chart's mail settings are off), so Nextcloud sends no email until it is
configured in the Nextcloud administration settings or in `values.j2.yml`.

To change a credential after a deployment exists, change it in the database/Nextcloud first and
then update the rendered value; changing only `values.j2.yml` on an existing installation does
not rotate a password already stored in the database.

## Parameters

None. Storage sizes (20Gi for Nextcloud, 8Gi for PostgreSQL) are fixed in `values.j2.yml`;
edit them there if a deployment needs different sizes (a PVC cannot be shrunk later).

## Decisions

- **PostgreSQL instead of the chart's default SQLite**, because Nextcloud is used concurrently and
  SQLite is not a supported production database. PostgreSQL is the chart's native subchart and
  gives a supported dump backup. MariaDB was not chosen to keep one database implementation and
  because Nextcloud's own documentation recommends PostgreSQL for larger installations.
- **No Redis.** With one app pod, Nextcloud's APCu local cache and the database locking are
  enough. There is no distributed file locking or session cache, so the driver marks `app`
  `singleton: true`: it must not be scaled to several replicas. Redis can be added later for a
  scale-out installation.
- **Cron through the chart's sidecar** (`cronjob.enabled: true`, `type: sidecar`) instead of a
  Kubernetes CronJob: the sidecar shares the app's ReadWriteOnce volume without scheduling a
  second pod on the same node. `/cron.sh` runs as root in the same image, which is the chart's
  documented mode for the sidecar.
- **Backups cover both kinds of state:** the file/app/config volume as a disk backup and the
  database as a `postgres` dump. `/backup` is an `emptyDir` volume mounted at `/backup` in the
  PostgreSQL container, as the Styrmin dump strategy requires.
- **Ingress only serves `primary_fqdn`.** Chart 9.4.1 has exactly one host value
  (`nextcloud.host`) and no extra-host or additional-Ingress value, so `human_readable_fqdn` is
  not served. Adding a second hand-written Ingress was avoided because it would bypass the
  chart's host/probe configuration.
- **Reverse proxy configuration is added explicitly.** The chart enables
  `reverse-proxy.config.php` by default but sets neither `trusted_proxies` nor
  `overwriteprotocol`. `nextcloud.configs` therefore writes them (plus `overwrite.cli.url`), so
  HTTPS terminated at the Ingress produces correct URLs and cron links. The trusted proxy ranges
  are the common private ranges (`10/8`, `172.16/12`, `192.168/16`, `100.64/10`); narrow them if
  the cluster's ingress controller uses other addresses.
- **Startup probe** with a 260 s budget, so the first installation into an empty database is not
  killed by liveness before Nextcloud answers `/status.php`.
- **Egress to the internet is allowed**, because Nextcloud uses it for the app store, update
  checks, previews and (when configured) SMTP. Egress to the deployment's own pods and DNS is
  allowed as well.
- **`nginx.ingress.kubernetes.io/proxy-body-size: "0"`** avoids the 1 MiB default upload limit of
  the nginx ingress controller. Other controllers, such as Traefik, ignore the annotation; their
  own body limit must be raised if they have one.
- **Image tag `<app version>-apache`**: the official Nextcloud registry tags carry an Apache/FPM
  suffix, and the chart's default image is the Apache variant. The range is restricted to the
  35.0 patch series because that is the series chart 9.4.1 ships and was checked with.

## Checked

`python3 scripts/check_driver.py nextcloud-driver --check-images` reports **0 FAIL, 0 WARN**:

- the spec loads, the template renders, and `helm template` renders 15 resources;
- both components match their workloads and containers, the external service is served on its
  primary host, every pod reaches DNS, and both backup declarations are valid;
- both image tags exist for linux/amd64 and linux/arm64;
- `--app-version 35.0.0` and `35.0.1` also render and their `-apache` tags exist.

## Not verified

- The driver was **not deployed on a Kubernetes cluster**: no first installation, login, upload,
  backup/restore, or upgrade (35.0.0 -> 35.0.2) was exercised.
- The PostgreSQL 17 version and the `bitnamilegacy` images are the subchart's defaults; they were
  only checked for existence, not run against Nextcloud 35.
- The trusted-proxy ranges and the nginx upload annotation assume a typical private cluster /
  nginx-ingress setup; other ingress controllers may need different values.
- The `human_readable_fqdn` URL is intentionally not served.
