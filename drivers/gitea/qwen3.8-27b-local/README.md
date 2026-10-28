# gitea driver for Styrmin

Deploys **Gitea**, a self-hosted Git service, with the official Gitea Helm chart. It runs the Gitea
web/SSH server (`docker.gitea.com/gitea:<version>-rootless`) with a single-writable **PostgreSQL 17**
database (Bitnami chart) and an egress NetworkPolicy for the deployment namespace. The server,
database and ingress hosts are wired together with the fixed Styrmin `styrmin-*` names.

| Component | Workload | Backup |
|---|---|---|
| `server` | Deployment/styrmin-gitea | disk (volume `data`) |
| `postgresql` | StatefulSet/styrmin-postgresql | postgres (database `gitea`) |

- Chart: `https://dl.gitea.com/charts/gitea` 12.7.0 (application 1.27.0, supported `>=1.24,<1.28`).
- Images: `docker.gitea.com/gitea:{{ app.version }}-rootless` (app version tag, chart appends `-rootless`),
  `docker.io/bitnamilegacy/postgresql:17.6.0-debian-12-r4` (subchart default).

## Load and deploy

```bash
styrminctl drivers load-version <git-url> <branch> --driver-directory <path/to/gitea-driver>
styrminctl deployments create gitea 1.27.0 <environment> --cluster <cluster>
```

Open the deployment's primary URL (and human-readable URL); both are on the Ingress and Gitea
advertises the primary host as its DOMAIN/ROOT_URL.

## Credentials

All are fixed values in `values.j2.yml` (never generated). To change one, edit the value there and
reload the driver version.

| What | Value | Where |
|---|---|---|
| Gitea admin login | user `admin`, password `GiteaAdmin-2026!`, email `admin@localhost` | `gitea.admin` |
| PostgreSQL superuser `postgres` | `GiteaPostgres-2026!` | `postgresql.global.postgresql.auth.postgresPassword` |
| Gitea DB user `gitea` (database `gitea`) | `GiteaDb-2026!` | `postgresql.global.postgresql.auth.password` |

The Gitea chart reads `postgresql.global.postgresql.auth.*` and fills its own database settings from
it, so one value serves both the database and the application (Bitnami global values take precedence
over the chart-local `postgresql.auth.*`, which are set to the same values here).

## Parameters

None.

## Decisions

- **Plain `postgresql` subchart, not the chart default `postgresql-ha`:** Gitea needs one writable
  database; the repmgr cluster adds pgpool and witness pods Styrmin has no components for and is
  overkill. Chosen at scaffold time (`--set postgresql-ha.enabled=false --set postgresql.enabled=true`).
- **Valkey disabled (`valkey.enabled=false`, `valkey-cluster.enabled=false`):** the chart rebuilds the
  queue/session/cache connection string from the 44-character release name
  (`gitea.inline_configuration.defaults` in `_helpers.tpl`), which cannot be pointed at the
  `styrmin-valkey-*` services that `fullnameOverride` creates. With both subcharts disabled the chart
  falls back to in-memory session/cache and the `level` queue, which Gitea runs fine on a single
  replica. Sessions therefore do not survive a pod restart; that is the trade-off.
- **`gitea.config.database.HOST: "styrmin-postgresql:5432"` set explicitly:** the chart's default
  builds `<release>-postgresql` from the release name, but `fullnameOverride` names the Service
  `styrmin-postgresql`; the chart only applies its default when `HOST` is empty, so the override sticks.
- **`gitea.config.server.PROTOCOL` from `instance.config.external_url_scheme`:** makes ROOT_URL match
  the deployment's URL scheme; the chart itself takes DOMAIN from the first ingress host.
- **Fixed credentials** as listed above, instead of the chart's random-looking defaults.
- **`test.enabled: false`:** the chart's busybox pod is a Helm `test-success` hook only (never deployed);
  disabling keeps the release's image list clean.
- **No internet egress** in the NetworkPolicy: Gitea serves local git traffic; allowing it would only
  matter for external webhooks/OAuth providers, which is intentionally not granted. DNS plus
  namespace-internal pod traffic is allowed for every pod (Styrmin denies all egress otherwise).
- **`supported_versions: ">=1.24,<1.28"`:** the chart ships app 1.27.0; the `-rootless` image tags for
  1.24.0, 1.25.0, 1.26.0 and 1.27.0 all exist on `docker.gitea.com` (verified against the registry).

## Checked

`check_driver.py --check-images`: **0 FAIL, 0 WARN** (default 1.27.0). Both images verified to exist
for linux/amd64 and linux/arm64. Rendering with `--app-version 1.26.0`: 0 FAIL, 0 WARN.
The rendered Gitea config (secret `styrmin-gitea-inline-config`) was inspected: DB host
`styrmin-postgresql:5432`, user/password/database `gitea`, memory session/cache, level queue,
DOMAIN/ROOT_URL from the primary FQDN, PROTOCOL from the external URL scheme.

## Not verified

Not deployed on a real cluster: runtime behavior (admin login, git push/clone over HTTPS and SSH,
level-queue and in-memory session behavior across restarts) was not tested, only rendered manifests
and registry existence were checked.
