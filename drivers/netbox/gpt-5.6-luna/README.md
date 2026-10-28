# NetBox Styrmin driver

This driver deploys NetBox, the network source of truth, with the official
`netbox/netbox` Helm chart 8.3.97 and NetBox 4.7.3. It includes:

| Component | Workload/container | Storage or backup |
|---|---|---|
| `server` | Deployment `styrmin-netbox`, `netbox`, port 8080 | `disk` backup of the `media` volume |
| `worker` | Deployment `styrmin-netbox-worker`, `netbox-worker` | none |
| `database` | PostgreSQL StatefulSet `styrmin-postgresql`, `postgresql` | `postgres` dump plus backup volume |
| `cache` | Valkey StatefulSet `styrmin-valkey-primary`, `valkey` | none; cache is rebuildable |

The external `netbox` service exposes the chart Ingress on both the primary and human-readable
Styrmin hostnames. PostgreSQL and Valkey are bundled subcharts. Valkey is configured as a single,
non-persistent instance because NetBox can rebuild this task/cache data; PostgreSQL and media hold
the durable application data.

## Choices and credentials

- The newest chart available from `https://charts.netbox.oss.netboxlabs.com/` was pinned at 8.3.97,
  whose chart metadata declares appVersion 4.7.3. The supported range is limited to the 4.7 minor
  series because that is the application version verified against this chart.
- Bitnami PostgreSQL and Valkey images use `bitnamilegacy` with explicit tags. This avoids the
  unversioned/current Bitnami images and makes rendered image checks reproducible.
- The chart's worker `waitForBackend` init container is disabled. It uses `kubectl` against the
  Kubernetes API, which would require an additional egress exception; Helm already orders the
  workloads and the worker can retry its application connections.
- The chart's default random secret key, API-token pepper, database passwords, cache password,
  superuser password, and API token are replaced with fixed template values so restored/cloned
  deployments remain consistent. Change all `change-before-production` values in
  `values.j2.yml` before a real deployment. The default login is `admin` with password
  `netbox-admin-password-change-before-production`.
- Internet egress is not granted. The shipped NetworkPolicy allows DNS and traffic to pods in the
  deployment namespace. Enable and document an additional policy if plugins or webhooks need
  external endpoints.
- The worker mounts the same durable `media` PVC as the server, so the checker may report that the
  worker has persistent storage without its own backup block. The server's `disk` backup covers
  that shared PVC; a second worker backup would duplicate the snapshot.

## Load and deploy

```bash
styrminctl drivers load-version <git-url> main --driver-directory netbox-driver
styrminctl deployments create netbox 4.7.3 <environment> --cluster <cluster>
```

## Verification

The skill's offline checker was run as follows (the environment did not have `uv`):

```bash
python3 /root/.config/opencode/skills/creating-styrmin-drivers/scripts/check_driver.py \
  /work/netbox-driver --check-images --keep /work/netbox-driver/.check
```

The checker validates the driver schema, rendered Helm manifests, component labels and containers,
Ingress hosts, resource names, egress policy, backups, and image availability for amd64/arm64.
It reported **0 FAIL** and **1 WARN**. The warning is expected: the worker mounts the server's
shared `media` PVC, whose single `disk` backup is declared on `server`; a second worker snapshot
would duplicate it. Cluster deployment, first-start migration timing, backup/restore, upgrades, and
real login have not been verified here.
