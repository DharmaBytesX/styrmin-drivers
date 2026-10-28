# grafana driver for Styrmin

Deploys [Grafana](https://github.com/grafana/grafana), the observability dashboard platform, with
the official `grafana-community/grafana` Helm chart 14.2.0 (application 13.3.0). The chart runs one
`grafana` container; Grafana stores its SQLite database, dashboards and plugins on a persistent
volume (`/var/lib/grafana`). The chart's Ingress exposes Grafana on the driver's external service
URL.

| Component | Workload | Backup |
|---|---|---|
| `server` | Deployment `styrmin-grafana`, container `grafana`, port 3000 | `disk` (snapshot of the `storage` PVC) |

- Service `grafana` (external, HTTP): the chart's Ingress serves `primary_fqdn` and
  `human_readable_fqdn` at path `/`.
- Chart: `https://grafana-community.github.io/helm-charts/grafana` 14.2.0 (application 13.3.0).
- Images: `docker.io/grafana/grafana:13.3.0-distroless`, plus the chart's `library/busybox:1.38.0`
  `init-chown-data` init container.

## Credentials

- Grafana admin user `admin` / password `grafana-admin-password` (fixed, set in `values.j2.yml` via
  `adminUser` / `adminPassword`; the chart stores them in the `styrmin-grafana` Secret). A generated
  password would break restored or cloned deployments. Change both in `values.j2.yml` before a real
  deployment. Grafana may still prompt to change the password on first login.

## Parameters

None.

## Decisions

- **Persistence enabled** (`persistence.type: pvc`, storage class from
  `{{ instance.config.standard_storage.storage_class }}`, `10Gi`): Grafana is stateful; without a
  PVC its dashboards, datasources and users are lost on restart. The size `10Gi` is the chart's
  default.
- **`disk` backup of the `storage` volume**: Grafana keeps its SQLite database in the `storage` PVC,
  so a Velero volume snapshot backs it up. There is no SQL-dump backup strategy for SQLite.
- **Image tag `{{ app.version }}-distroless`**: `docker.io/grafana/grafana` tags releases with a
  `-distroless` suffix (the chart's default is `<appVersion>-distroless`), verified by
  `--check-images` for `13.3.0-distroless`.
- **`server.root_url` set from the external URL**: the chart already sets `server.domain` from the
  first ingress host, but Grafana would otherwise compute a base URL with the internal `:3000` port;
  setting `root_url` makes redirects and generated links use the externally reachable URL.
- **Internet egress allowed**: Grafana downloads plugins from grafana.com, contacts external data
  sources, and checks for updates, so the NetworkPolicy allows the internet (without private
  ranges), DNS and the namespace's pods.
- **`supported_versions: ">=13.3,<13.4"`**: the chart 14.2.0 ships Grafana 13.3.0; the range is
  pinned to the 13.3 minor line this chart tracks.
- **No actions**: the chart installs, upgrades and uninstalls Grafana on its own.

## Checked

`check_driver.py --check-images`: **0 FAIL, 1 WARN**. The WARN is
`grafana.ini.server.root_url is not a value of the chart`, which is a false positive: the chart's
`grafana.ini` map accepts arbitrary keys (`additionalProperties: true`) and renders every key
(including `[server] root_url`) into the `grafana.ini` ConfigMap — confirmed in the rendered
manifests. Images `grafana:13.3.0-distroless` and `busybox:1.38.0` both resolve for linux/amd64 and
linux/arm64.

## Not verified

Not deployed on a cluster: startup time, the login flow, and backup/restore were not exercised.

```bash
styrminctl drivers load-version <git-url> main --driver-directory grafana-driver
styrminctl deployments create grafana 13.3.0 <environment> --cluster <cluster>
```
