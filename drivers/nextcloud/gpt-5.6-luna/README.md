# Nextcloud driver for Styrmin

This driver deploys Nextcloud 35.0.2 with the official Nextcloud Helm chart 9.4.1. It provides a
single Apache web deployment, persistent Nextcloud application/data storage, a cron sidecar for
background jobs, and a standalone persistent MariaDB database with Styrmin backup support.

| Component | Workload | Backup |
|---|---|---|
| `app` | Deployment/styrmin-nextcloud | Nextcloud persistent volume snapshot |
| `database` | StatefulSet/styrmin-mariadb | MySQL dump and volume snapshot |

## Load and deploy

```text
styrminctl drivers load-version <git-url> <branch> --driver-directory nextcloud-driver
styrminctl deployments create nextcloud 35.0.2 <environment> --cluster <cluster>
```

The external service is `nextcloud`; Styrmin supplies its primary and human-readable hostnames.

## Credentials

The initial Nextcloud administrator is `admin` / `nextcloud-admin-password`. The MariaDB account
is `nextcloud` / `nextcloud-db-password`. These fixed values are in `values.j2.yml` so restores
and cloned deployments retain access. Change them before production use by editing the driver and
keep the database password identical in `externalDatabase.password` and `mariadb.auth.password`.

## Parameters and decisions

- Parameters: none. Storage uses the deployment's standard storage class.
- MariaDB is enabled instead of the chart's default SQLite because SQLite and the default
  `emptyDir` storage are not durable or appropriate for concurrent file-sharing workloads.
- MariaDB is standalone and marked singleton because a second database primary would not share
  the same data.
- The chart's Apache image is retained; its `image.tag` is the application version and the chart
  appends the `-apache` flavor suffix.
- The chart exposes one ingress host, so the primary hostname is used. The human-readable hostname
  is included in Nextcloud's trusted domains but cannot be added as a second chart ingress rule.
- An egress policy permits DNS and traffic between deployment pods. Internet egress is not enabled
  because this base deployment does not require it to start; optional external integrations may
  require a later policy adjustment.

## Verification

`check_driver.py /work/nextcloud-driver --check-images` passed with 0 FAIL and 0 WARN. The offline
check rendered the Helm chart and validated the driver schema, workloads, services, ingress,
component labels, image tags, backups, and DNS egress policy.

Not verified: deployment on a Kubernetes cluster, ingress-controller behavior, persistent-volume
provisioning, and live Nextcloud login or backup restoration.
