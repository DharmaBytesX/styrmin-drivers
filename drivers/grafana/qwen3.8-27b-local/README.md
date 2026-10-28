# grafana driver for Styrmin

Deploys Grafana, the tool for querying, visualizing and alerting on time series
metrics, as a single-pod Deployment with an embedded SQLite database on a
persistent volume. One external `grafana` HTTP service is exposed on both the
machine-readable and the human-readable host.

| Component | Workload | Backup |
|---|---|---|
| `server` | Deployment/styrmin-grafana | disk: PVC mounted at /var/lib/grafana |

- Chart: `https://grafana-community.github.io/helm-charts/grafana` 14.2.0 (application 13.3.0).
- Credentials: admin user `admin`, password `grafana-admin-password` by default.
  Choose a different one at creation time with the `admin_password` parameter, or
  change it after first login in the Grafana UI (Administration → Users → admin).
- Parameters: `admin_password` (string, default `grafana-admin-password`).
- Decisions:
  - `adminPassword` is a fixed value from the `admin_password` parameter, not the
    chart's generated `randAlphaNum 40` password: a restored or cloned deployment
    must still work with its existing data.
  - `image.tag` is `{{ app.version }}-distroless`: the chart's default image is the
    distroless variant, and the tag is made to follow the deployed app version
    (verified to exist for linux/amd64 and linux/arm64).
  - `rbac.create: false`: no container in this deployment calls the Kubernetes API
    (all chart sidecars are disabled), so the chart's Role/ClusterRole and binding
    are not needed; removing them also avoids cluster-scoped objects.
  - `testFramework.enabled: false`: the chart's helm-test bats jobs are not needed.
  - `persistence.enabled: true` with the cluster's standard storage class: the
    embedded SQLite database, dashboards and plugins live under `/var/lib/grafana`.
  - Component `server` is `singleton: true`: the database is embedded, so a second
    pod would not share it.
  - Backup is `kind: disk` on the `storage` volume: it is the only non-rebuildable
    data; the `search` volume (unified storage index) is an emptyDir that is
    rebuilt when missing.
  - A NetworkPolicy allows egress to DNS and to the other pods of the deployment
    only, with no blanket internet access. Consequence: installing plugins from
    grafana.com will not work; dashboards and data sources inside the cluster
    (or the same namespace) are unaffected.
  - `supported_versions` is pinned to `13.3.0`: the chart 14.2.0 is built for
    Grafana 13.3.0 and only the `13.3.0-distroless` image tag exists.
  - The ingress is created for both hosts with no ingress class: Styrmin assigns
    the class. `root_url`/`server.domain` are left to the chart defaults.
- Checked: `check_driver.py --check-images` passes with 0 FAIL, 0 WARN;
  `grafana/grafana:13.3.0-distroless` and `busybox:1.38.0` exist for
  linux/amd64 and linux/arm64.
- Not verified: not deployed on a cluster; login, dashboard creation and backups
  were not exercised end-to-end; plugin installation from grafana.com is not
  expected to work without internet egress.
