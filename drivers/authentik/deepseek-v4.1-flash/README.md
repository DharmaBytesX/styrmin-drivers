# authentik driver for Styrmin

Deploys [authentik](https://goauthentik.io), an open-source identity provider, from its official Helm
chart: the `server` Deployment (HTTP 9000, web UI/API/SSO endpoints), the `worker` Deployment
(background tasks, outposts, migrations), and a bundled PostgreSQL StatefulSet. The server is exposed
through the chart's Ingress on both the primary and the human-readable host.

| Component | Workload | Image | Backup |
|---|---|---|---|
| `server` | Deployment/styrmin-authentik-server | `ghcr.io/goauthentik/server:2026.8.5` | none |
| `worker` | Deployment/styrmin-authentik-worker | `ghcr.io/goauthentik/server:2026.8.5` | none |
| `postgresql` | StatefulSet/styrmin-postgresql | `docker.io/library/postgres:17.11-trixie` | postgres dump |

- Chart: `https://charts.goauthentik.io/authentik` 2026.8.5 (application 2026.8.5).
- Supported application versions: `>=2026.8,<2026.9`; default `2026.8.5`.
- The image tag follows `{{ app.version }}` in `global.image.tag`.

## Load and deploy

```bash
styrminctl drivers load-version <git-url> <branch> --driver-directory authentik-driver
styrminctl deployments create authentik 2026.8.5 <environment> --cluster <cluster>
```

## Credentials

All credentials are fixed in `values.j2.yml`, so a restore or clone matches the data:

| Purpose | Default | Where to change |
|---|---|---|
| First admin user | `admin@example.com` / `styrmin-authentik-admin-password` | `authentik.bootstrap_email`, `authentik.bootstrap_password` |
| authentik signing key | fixed 50+ character key | `authentik.secret_key` (do not change after first install) |
| Application database password | `styrmin-authentik-database-password` | `authentik.postgresql.password` and `postgresql.auth.password` (keep equal) |
| PostgreSQL superuser password | `styrmin-authentik-postgres-superuser-password` | `postgresql.auth.postgresPassword` |
| PostgreSQL replication password | fixed, unused while replicas are off | `postgresql.auth.replicationPassword` |

Change the values before the first deployment; after the data exists, change the password in the
database as well as here. The bootstrap admin values are used only on an empty data directory
(first install); the user is created with this password, so change it in the UI after login.

Parameters: none.

## Decisions

- **Official chart, pinned** `2026.8.5`; application and chart versions are the same upstream, so the
  supported range stays inside the 2026.8 minor.
- **Bundled PostgreSQL subchart** (`postgresql.enabled: true`) rather than an external database, so
  the driver is self-contained. The chart defaults the app host to `<release>-postgresql`, which is
  wrong under Styrmin's long release name; `authentik.postgresql.host` is set to the rendered
  Service `styrmin-postgresql`. The application password is the same in `authentik.postgresql.password`
  and `postgresql.auth.password`.
- **No Redis**: authentik removed the Redis dependency in 2025.10 (everything uses PostgreSQL) and
  the chart has no Redis subchart.
- **Managed outposts disabled** (`serviceAccount.create: false`). The chart's service account and
  RBAC exist for outposts that authentik deploys into Kubernetes; Styrmin's egress policy blocks the
  Kubernetes API, so those outposts cannot work. Core server/worker need no API access. External
  outposts (proxy, LDAP, RADIUS, ...) must be run outside this deployment; outpost containers need
  network access to the authentik server.
- **Bootstrap admin** via `AUTHENTIK_BOOTSTRAP_EMAIL` / `AUTHENTIK_BOOTSTRAP_PASSWORD`, so the URL
  answers with a usable login instead of a setup page. The chart renders any key under `authentik.*`
  as `AUTHENTIK_*` in its Secret; `bootstrap_email` and `bootstrap_password` are not listed in the
  chart's `values.yaml`, which is why the check reports them as unknown, but the rendered Secret
  contains both variables (verified in the rendered manifests).
- **Ingress**: the chart's own Ingress with both `primary_fqdn` and `human_readable_fqdn`, default
  path `/` and `Prefix`. No ingress class is set; Styrmin sets it.
- **First start is slow**: the server/worker run database migrations. The chart's startup probe
  allows up to 10 minutes; normally readiness comes well within Styrmin's 270 s, but a first install
  with a large import may exceed it and turn the deployment to running later.
- **Egress**: DNS and other pods of the deployment only (authentik has no runtime internet
  requirement; email, GeoIP and external outposts are not configured).

## What was checked

`check_driver.py --check-images`:

- 0 FAIL.
- WARN `authentik.bootstrap_email` / `authentik.bootstrap_password` are not values of the chart:
  explained above; they are valid authentik configuration keys and appear in the rendered Secret.
- `helm template` rendered 15 resources; components match their workloads and containers; the
  external service is reachable through both ingress hosts; DNS egress is allowed; the PostgreSQL
  backup volume is mounted at `/backup`; both images exist for linux/amd64 and linux/arm64.

## Not verified

- Not deployed on a cluster; no live login, SSO flow, or first-boot timing was tested.
- Backup dump/restore was validated only structurally (volume and PostgreSQL env), not executed.
- External outposts, email/SMTP, GeoIP and Kubernetes-managed outposts are out of scope.
