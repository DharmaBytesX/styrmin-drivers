# Gatus driver for Styrmin

Deploys [Gatus](https://github.com/TwiN/gatus), the automated service health dashboard, as a single
Deployment with its own Ingress, a ConfigMap holding Gatus' configuration, a PersistentVolumeClaim
for the SQLite database that keeps the uptime history, and an egress NetworkPolicy.

| Component | Workload | Backup |
|---|---|---|
| `server` | Deployment `styrmin-gatus`, container `gatus`, port 8080, singleton | `disk` (snapshot of the `data` volume: the SQLite database) |

- Chart: `oci://ghcr.io/bjw-s-labs/helm/app-template` 5.2.1 (application 5.34.0, image `twinproduction/gatus:v5.34.0`).
  The official Gatus chart (`https://twin.github.io/helm-charts/gatus` 1.5.0) is **not** used: see
  "Decisions" below.
- Service `gatus` (external): the Ingress serves `primary_fqdn` and `human_readable_fqdn`, path `/`.
  The URL is `{{ instance.config.external_url_scheme }}://<primary_fqdn>`.
- Network: a NetworkPolicy shipped by the chart allows DNS, the pods of the environment, the pods of
  the deployment's own namespace and the internet without the private ranges. Gatus watches
  endpoints over HTTP, so it needs the internet.
- Credentials: none. Gatus has no authentication by default, and the driver does not enable one.
  See "Adding authentication" below.
- Parameters:
  - `monitored_url` (string, default `http://styrmin-gatus:8080/`): the URL shown as the `primary`
    row of the dashboard. The `gatus` row always watches the instance's own `/health` endpoint.

## Load and deploy

```bash
styrminctl drivers load-version <git-url> <branch> --driver-directory gatus-driver
styrminctl deployments create gatus 5.34.0 <environment> --cluster <cluster>
```

## What the dashboard shows

The ConfigMap `styrmin-gatus` holds `config.yaml`, mounted read-only at `/config` and read through
`GATUS_CONFIG_PATH=/config/config.yaml`:

```yaml
endpoints:
  - name: gatus
    url: "http://styrmin-gatus:8080/health"
    interval: 30s
    conditions:
      - "[STATUS] == 200"
  - name: primary
    url: "http://styrmin-gatus:8080/"
    interval: 30s
    conditions:
      - "[STATUS] == 200"
storage:
  type: sqlite
  path: /data/data.db
```

To watch more services, add entries to `endpoints` in `values.j2.yml` (the file that holds the
ConfigMap data) and redeploy. Gatus' configuration reference is in the
[Gatus README](https://github.com/TwiN/gatus#configuration).

## Decisions and why

- **The bjw-s app-template chart instead of the official `twin/gatus` chart.** Styrmin denies all
  egress in a deployment namespace, DNS included. The official chart has no `extraDeploy` /
  `extraObjects` / `extraManifests` value, and its own `networkPolicy` value renders ingress rules
  only, so it cannot carry the egress policy the deployment needs: `check_driver.py` on the
  official chart reports `FAIL [network] ... cannot reach DNS (kube-dns, UDP 53)`. Gatus is a
  single-container application, which is exactly the case the skill's `references/networking.md`
  sends to the app-template chart. The upstream chart stays the reference for the image name, the
  tag format (`v<version>`), the configuration format, the probes (`/health` on 8080) and the
  security context (uid/gid 65534, read-only root filesystem), all of which the driver reproduces.
- **SQLite storage on a PersistentVolumeClaim.** Gatus stores the uptime history of the endpoints;
  with its default in-memory storage that history is lost at every restart. The volume is
  `ReadWriteOnce`, so the component is `singleton: true` and the Deployment uses the `Recreate`
  strategy (the old pod must release the volume first). `fsGroup: 65534` lets the non-root
  container write the database file.
- **`disk` backup of the `data` volume.** The database is the only state of the application, so
  Styrmin snapshots it with Velero like any other volume.
- **Probes on `/health`.** Gatus registers that route itself (`api/api.go`) and answers 200 with a
  JSON body; the official chart uses it too. The startup probe allows 270 s (54 × 5 s), the limit
  Styrmin gives a pod to become Ready.
- **Non-root, read-only root filesystem.** The same values as the official chart
  (`runAsNonRoot`, uid/gid 65534, `readOnlyRootFilesystem`), so the container only writes to the
  data volume.
- **Egress to the internet.** It is the purpose of the application: the dashboard watches HTTP
  endpoints, which are usually outside the cluster. The private ranges stay excluded, so the pods
  cannot reach the cluster's own services, Styrmin's server included.
- **Version range `>=5.34,<5.35`.** 5.34.0 is the version the official chart 1.5.0 ships, and the
  only 5.34.x tag that exists in the registry. Widen the range only after checking the new tags
  with `check_driver.py --check-images --app-version <version>`.

## Adding authentication

Gatus' `security` section is not enabled by the driver (the official chart does not enable it
either). To protect the dashboard, add to the `config.yaml` of the ConfigMap in `values.j2.yml`,
for example:

```yaml
security:
  basic:
    username: "admin"
    password-bcrypt-base64: "JDJhJDEwJHRiMnRFakxWazZLdXBzRERQazB1TE8vckRLY05Yb1hSdnoxWU0yQ1FaYXZRSW1McmladDYu"
```

Gatus wants the password hashed with bcrypt and encoded in base64 (cost 9 is a good default), for
example with `htpasswd -bnBC 9 "" 'change-me' | tr -d ':\n'`. The username and the hash then sit
in clear text in `values.j2.yml`: change them before a real deployment.

## Checked

`check_driver.py <driver-dir> --check-images`: **0 FAIL, 0 WARN**. The checks cover the driver
schema, the rendering of `values.j2.yml`, `helm template` of the pinned chart, the component
labels and container name, the external service and its Ingress hosts and paths, the DNS egress
policy, the backup declaration and the image tag (`twinproduction/gatus:v5.34.0` exists for
linux/amd64 and linux/arm64). The same run with `--param monitored_url=https://example.org:8443/health`
confirms that the parameter reaches the rendered configuration.

The application itself was also run locally: the `gatus` binary of `twinproduction/gatus:v5.34.0`
was extracted from the image and started with the configuration this driver renders, as the
unprivileged user the driver runs it as (uid/gid 65534) and with a data directory writable by that
group (what `fsGroup: 65534` gives on the volume). Gatus accepted the configuration
("Validated 2 endpoints"), monitored both endpoints (`success=true`), answered `GET /health` with
`200 {"status":"UP"}` (the probe of the Deployment) and `GET /` with `200` (the dashboard), and
created the SQLite database (`data.db`, `data.db-shm`, `data.db-wal`) in the data directory.

## Not verified

- The driver was **not deployed on Kubernetes**: no pod was started in a cluster, no URL was opened
  through the Ingress, and the NetworkPolicy was not enforced by a real CNI. The rendering, the
  image tag and the network policy are checked offline only.
- The PVC, the `fsGroup` ownership of the volume and the `Recreate` strategy were not exercised on a
  cluster (the local run emulated the permissions with a group-writable directory).
- Backup and restore were not exercised (the `disk` backup is a Velero snapshot of the `data`
  volume).
- The upgrade path was not exercised.
- Only version 5.34.0 was rendered and checked.
