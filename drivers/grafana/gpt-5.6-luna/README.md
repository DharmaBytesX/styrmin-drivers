# grafana driver for Styrmin

This driver deploys Grafana's web server from the Grafana Community Helm chart,
with an external HTTP ingress on the two Styrmin-provided Grafana hostnames.
Grafana data is stored in a 10Gi persistent volume using Styrmin’s standard
storage class. The deployment is intentionally single-instance because the
chart uses a `ReadWriteOnce` volume and the default Grafana database is SQLite.

| Component | Workload | Backup |
|---|---|---|
| `server` | Deployment/styrmin-grafana | `disk` snapshot of the `storage` volume |

- Chart: `https://grafana-community.github.io/helm-charts/grafana` 14.2.0 (application 13.3.0).
- Image: `docker.io/grafana/grafana:{{ app.version }}-distroless`.
- Credentials: the administrator is `admin` with password `GrafanaAdmin-ChangeMe-13.3.0`.
  Change `adminPassword` in `values.j2.yml` before loading the driver for a deployment.
  The Grafana signing key is also fixed in `env.GF_SECURITY_SECRET_KEY`; change it
  together with the password when creating a new deployment.
- Parameters: none.
- Decisions: the chart's application version range is limited to `>=13.3,<13.4` because
  this driver is authored and checked against chart 14.2.0 shipping Grafana 13.3.0.
  The distroless image suffix follows the chart default. Persistence is enabled for
  dashboards and SQLite data. The persistent volume is backed up as a disk snapshot.
  Internet egress is not allowed because this base deployment
  does not install plugins or fetch dashboards; DNS and same-namespace pod egress remain
  allowed for configured internal data sources.
- Checked: `check_driver.py --check-images` completed with 0 FAIL and 0 WARN.
- Not verified: the driver was not loaded into Styrmin or deployed to a Kubernetes cluster;
  restore and upgrades were not exercised.
