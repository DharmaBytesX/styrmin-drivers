# authentik driver for Styrmin

Deploys authentik, the identity provider, from the official chart: a server (web and API, port 9000), a worker, and a PostgreSQL database from the chart's Bitnami subchart. Authentik 2026.8 no longer needs Redis, so the driver has no cache component.

| Component | Workload | Backup |
|---|---|---|
| `server` | Deployment/styrmin-authentik-server | none |
| `worker` | Deployment/styrmin-authentik-worker | none |
| `postgresql` | StatefulSet/styrmin-postgresql | postgres dump of database `authentik` |

- Chart: `https://charts.goauthentik.io/authentik` 2026.8.5 (application 2026.8.5).
- Images: `ghcr.io/goauthentik/server:2026.8.5` (server and worker), `docker.io/library/postgres:17.11-trixie` (the chart's own default).
- Supported versions: `>=2026.8,<2026.9`. Reason: only 2026.8.5 was rendered and checked, and authentik can change its settings between releases.

## Load and deploy

```bash
styrminctl drivers load-version <git-url> <branch> --driver-directory authentik-driver
styrminctl deployments create authentik 2026.8.5 <environment> --cluster <cluster>
```

## Credentials

Fixed values are in `values.j2.yml`. Change them before real use.

- Administrator: user `akadmin`, email `admin@example.com`, password `styrmin-admin-change-me` (`authentik.bootstrap_password`). authentik reads it only on first start; later changes are made in authentik.
- `authentik.secret_key`: cookie signing key. Do not change it after the first start.
- Database user `authentik`, password `styrmin-authentik-db-password`; PostgreSQL administrator `postgres`, password `styrmin-postgres-admin-password`.

## Parameters

None.

## Decisions

- Database subchart switched on with `postgresql.enabled=true`: the chart ships it disabled, and authentik needs a database.
- Bootstrap settings are set as `authentik.bootstrap_password` and `authentik.bootstrap_email`. The check script warns that the chart does not declare them. The warning is expected: the chart turns every key under `authentik` into an `AUTHENTIK_<KEY>` variable, and the rendered Secret contains `AUTHENTIK_BOOTSTRAP_PASSWORD` and `AUTHENTIK_BOOTSTRAP_EMAIL`.
- `serviceAccount.create: false`: the service account and its ClusterRole only serve managed outposts, which call the Kubernetes API (rule 16) and need cluster-wide rights. Managed (Kubernetes-integrated) outposts therefore do not work; run outposts manually instead.
- Internet egress allowed (private ranges excluded) because authentik contacts external identity providers, SMTP servers and webhooks. Remove that rule to block it.
- Fixed example passwords, because Styrmin requires fixed credentials. They are not secret: change them.

## Checked

`check_driver.py --check-images`: 0 FAIL, 2 WARN (the two bootstrap keys described above). Image tags exist for linux/amd64 and linux/arm64.

## Not verified

- Not deployed on a cluster: startup time within 270 s, the URL answering, and the bootstrap login are untested.
- Backup and restore of the database were not run.
- Other 2026.8.x versions were not rendered.
