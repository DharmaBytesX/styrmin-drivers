# nextcloud driver for Styrmin

This driver deploys Nextcloud, a file sharing and collaboration server, on Kubernetes. Users reach
Nextcloud at one URL, and their files live on a persistent volume. A separate PostgreSQL pod holds
the database. Source: [Nextcloud server](https://github.com/nextcloud/server).

| Component | Workload | Backup |
|---|---|---|
| `app` | Deployment/styrmin-nextcloud (container `nextcloud`) | disk snapshot of the `nextcloud-main` volume (`/var/www/html`) |
| `database` | StatefulSet/styrmin-nextcloud-postgresql (container `postgresql`) | `pg_dump` of the `nextcloud` database |

- Chart: `https://nextcloud.github.io/helm/nextcloud` 9.4.1, from the application's own repository
  on Artifact Hub (`nextcloud`, 190 stars). The chart's application version is 35.0.2.
- Images: `docker.io/library/nextcloud:35.0.2-apache` (the chart's default Apache flavor) and
  `docker.io/bitnamilegacy/postgresql:17.5.0-debian-12-r3` (the PostgreSQL subchart's image, from
  the `bitnamilegacy` mirror because Bitnami stopped publishing versioned tags under
  `docker.io/bitnami`). Both tags exist for linux/amd64 and linux/arm64, per `--check-images`.
- Subcharts: PostgreSQL 16.7.4 (enabled). MariaDB, Redis and Collabora are not deployed.

## Load and deploy

```bash
styrminctl drivers load-version <git-url> <branch> --driver-directory nextcloud-driver
styrminctl deployments create nextcloud 35.0.2 <environment> --cluster <cluster>
```

## Credentials

The driver sets fixed values, not generated ones. They are visible to anyone who can read the
driver, so change them before a production deployment by editing `values.j2.yml` and loading the
driver again.

| Account | Where | Default value |
|---|---|---|
| Nextcloud admin | `nextcloud.username`, `nextcloud.password` | user `admin`, password `nextcloud-admin-changeme` |
| Database user | `externalDatabase.password`, `postgresql.global.postgresql.auth.password` | user `nextcloud`, password `nextcloud-db-changeme` |

Keep the two database password values the same. The application reads the database password from
`externalDatabase.password`, and PostgreSQL sets its user password from the subchart value.

## Parameters

None. Sizes and storage class come from the driver and the deployment context.

## Decisions

- **Database: a PostgreSQL pod, not the chart's internal database.** The chart's default stores
  the database in the application's data volume. The driver uses a separate PostgreSQL pod so
  that the database has its own volume and its own `pg_dump` backup.
- **One URL.** The external service uses `services['nextcloud'].primary_fqdn` for
  `nextcloud.host`, which the chart uses for the Ingress and for `NEXTCLOUD_TRUSTED_DOMAINS`. The
  chart has no value for a second host, so the driver does not add `human_readable_fqdn`. The
  Ingress path is `/`.
- **HTTPS links.** `phpClientHttpsFix` is on, with `protocol` set from
  `instance.config.external_url_scheme`. The chart then sets `OVERWRITEPROTOCOL` to the scheme
  users reach. This makes Nextcloud generate links with the scheme users reach. I did not verify
  the chart's behavior beyond the rendered environment variable.
- **Image tag.** The tag is `{{ app.version }}-apache`, because Docker Hub tags carry the flavor
  suffix. The tag without the suffix is not used.
- **Storage.** The data volume is 20 GiB and the database volume is 8 GiB, both fixed in the
  driver. Both use the storage class from the context. Without `persistence.enabled`, the chart
  stores user files in an `emptyDir`, which is lost when the pod restarts.
- **Singletons.** Both components are `singleton: true`. The data volume is `ReadWriteOnce`, and
  the database has one primary with one data volume.
- **No cache.** The chart sets Redis environment variables only when `REDIS_HOST` is set, and the
  driver does not set it. Nothing else in the chart was checked for a cache requirement, so
  whether Nextcloud needs one for file locking is not verified.
- **Chart features off.** Collabora, the cron sidecar, the nginx front end, metrics, and S3 or Swift
  object storage stay at the chart's default (off). The cron sidecar runs `crond` as root in the
  application pod and is not needed for the first version.
- **No internet egress.** The egress NetworkPolicy allows DNS and traffic between the pods of the
  deployment. It does not allow the internet, so the Nextcloud app store and update checks do not
  work. Whether Nextcloud works fully without internet access is not verified. Add an `ipBlock`
  rule to `extraManifests` to open it.
- **Version range.** `supported_versions` is `>=35.0,<35.1`, which covers the chart's application
  version 35.0.2. Other 35.0.x versions were not rendered or checked.
- **Backups.** The application's disk snapshot and the database dump are taken separately, so
  they may not match at the same moment. The driver does not put Nextcloud into maintenance mode
  during a backup.
- **Icon.** The driver has no icon. The scaffold proposed one, but I could not confirm the URL
  points to the Nextcloud logo, so I left it out.

## Checked

`check_driver.py /work/nextcloud-driver --check-images`: 0 FAIL, 0 WARN.

- The spec follows the driver schema, and `values.j2.yml` renders to valid YAML.
- `helm template` renders the chart 9.4.1 with 14 resources.
- Components `app` and `database` match the Deployment and StatefulSet.
- The external service `nextcloud` is served on its primary FQDN.
- Every pod can reach DNS. The backup strategies `disk` and `postgres` match their components.
- Both images exist in their registries.

## Not verified

- Not deployed on a cluster. The rendered manifests were checked, not the running pods.
- Whether the pods become Ready within 270 seconds. Nextcloud installs itself on first start, and
  the install time was not measured.
- Login, file upload, sharing, and the URL in a browser.
- Restore from either backup.
- Other Nextcloud versions and other storage classes.
