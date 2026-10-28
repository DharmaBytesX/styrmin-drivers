# NetBox driver for Styrmin

Deploys NetBox (IPAM and DCIM) with the official `netbox` chart 8.3.97 (application 4.7.3)
and its Bitnami PostgreSQL and Valkey subcharts.

| Component | Workload | Backup |
|---|---|---|
| `server` | Deployment `styrmin-netbox`, container `netbox`, port 8080 | `disk` (media volume with uploads) |
| `worker` | Deployment `styrmin-netbox-worker`, container `netbox-worker` | none (no data of its own; shares volumes with `server`) |
| `postgresql` | StatefulSet `styrmin-postgresql`, container `postgresql`, singleton | `postgres` (pg_dump to `/backup`) |
| `valkey` | StatefulSet `styrmin-valkey-primary`, container `valkey`, singleton | none (ephemeral cache, persistence off) |

- Service `netbox` (external, container port 8080): the chart's Ingress serves
  `primary_fqdn` and `human_readable_fqdn`; no ingress class (Styrmin sets it).
- Images: app `ghcr.io/netbox-community/netbox:v4.7.3`, database
  `docker.io/bitnamilegacy/postgresql:17.6.0-debian-12-r4`, cache
  `docker.io/bitnamilegacy/valkey:8.1.3-debian-12-r3`, init container
  `docker.io/busybox:1.38.0`.
- Chart: `https://charts.netbox.oss.netboxlabs.com/netbox` 8.3.97; supported application
  versions `>=4.7,<4.8` (4.7.0 is the first release of the series; it requires PostgreSQL 15+,
  which the driver provides).

## Load and deploy

```bash
styrminctl drivers load-version <git-url> main --driver-directory netbox-driver
styrminctl deployments create netbox 4.7.3 <environment> --cluster <cluster>
```

## Credentials

All fixed (never generated); change them in `values.j2.yml` before a real deployment:

| What | Value |
|---|---|
| Superuser (username `netbox`) password | `netbox-superuser-password` |
| Superuser API token | `netbox-superuser-api-token-fixed-00000000000000000000000001` |
| Django `secretKey` | `netbox-django-secret-key-fixed-00000000000000000000000001` |
| API token pepper 1 | `netbox-api-token-pepper-1-fixed-0000000000000000000000001` |
| PostgreSQL password (user `netbox`, database `netbox`) | `netbox-postgresql-password` |
| Valkey password | `netbox-valkey-password` |

The database password is the same value in `externalDatabase.password` (the application's
settings) and `postgresql.auth.password` (the database itself).

Parameters: none.

## Decisions

- **Fixed, human-readable credentials** for every secret (rule 4), the same convention as the
  Semaphore worked example; the Django `secretKey` and the v2 API token pepper use 50-character
  values because NetBox rejects shorter ones.
- **PostgreSQL 17 image.** The chart's subchart is Bitnami PostgreSQL 18.x, which defaults to a
  PostgreSQL 18 image; `docker.io/bitnamilegacy` has no 18.x tags (newest is 17.6.0), so the
  image is pinned to 17. NetBox 4.7 requires PostgreSQL 15+, so 17 is fully supported.
- **Valkey standalone, no persistence.** The cache needs no replicas (that removes the replica
  StatefulSet; `replica.replicaCount: 0` would still render it) and no volume (cache data is
  ephemeral), so there is no backup for it.
- **Worker without the wait-for-backend init container** (`worker.waitForBackend.enabled: false`):
  that init container runs `kubectl` and calls the Kubernetes API, which Styrmin's network
  policies block. The server's `init-dirs` container (busybox `mkdir` only) needs no API access
  and stays.
- **allowedHosts** gets the two hosts Styrmin gives the service; the chart's readiness/startup
  probes send `allowedHosts[0]` as the `Host` header, so the primary FQDN must be first.
- **Egress NetworkPolicy** (shipped via `extraDeploy`) allows DNS and pod-to-pod in the namespace
  for every pod; there is no internet egress. The chart's release check is off by default and
  NetBox makes no other outbound calls at runtime.
- **No SMTP configuration**: the chart default (localhost:25) is left as-is; NetBox 4.7 raises an
  error only when mail is attempted without an `EMAIL` server configured, which the default
  satisfies structurally.

## Checked

`check_driver.py --check-images`: **0 FAIL, 1 WARN**, 29 resources rendered. The one WARN:
`docker.io/busybox:1.38.0` answered HTTP 401 to the script's registry query; the tag was checked
by hand and its manifest exists (verified with an anonymous pull token on
registry-1.docker.io, status 200), so the image is fine. The rendered manifests were inspected:
all six Secrets carry the expected keys, the projected `/run/secrets/netbox` volume maps the
PostgreSQL and Valkey passwords to the keys `configuration.py` reads (`db_password`,
`tasks_password`, `cache_password`), the Ingress hosts match `allowedHosts`, and the external
service port 80 targets the container's named port `http` (8080). `helm template` also renders
with `--app-version 4.7.0`.

## Not verified

- Not deployed on a cluster: no live readiness, no HTTP answer on the URL, no login tested.
- Backup and restore not run; the dump backup writes to an `emptyDir` mounted at `/backup` on the
  PostgreSQL StatefulSet.
- Upgrade (4.7.x to a later minor) not tested.
