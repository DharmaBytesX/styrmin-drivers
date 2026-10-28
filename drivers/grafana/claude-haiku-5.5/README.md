# Grafana driver for Styrmin

Grafana is the dashboard tool for querying and visualizing time series and metrics. This driver
deploys one Grafana server for a team, reached at the deployment's URL, with its data on a
persistent volume.

## What it deploys

- Component `server`: Deployment `styrmin-grafana`, container `grafana`, one pod (`singleton: true`).
- External service `grafana`: HTTP on port 3000, served through an Ingress at
  `{{ services['grafana'].primary_fqdn }}` and `{{ services['grafana'].human_readable_fqdn }}`.
- A PersistentVolumeClaim of 10 GiB on the environment's standard storage class, mounted at
  `/var/lib/grafana`. It holds the SQLite database, dashboards, users and plugins.
- An egress NetworkPolicy: DNS and traffic between the pods of this deployment only.
- Backup: a Velero disk snapshot of the data volume (`volume: storage`).
- Chart: `grafana` 14.2.0 from https://grafana-community.github.io/helm-charts/grafana
  ([Artifact Hub listing](https://artifacthub.io/packages/helm/grafana-community/grafana)).
- Images: `docker.io/grafana/grafana:<app version>-distroless` (13.3.0-distroless) and
  `docker.io/library/busybox:1.38.0` for the init container that sets the volume owner.
  The check script confirmed both tags exist for linux/amd64 and linux/arm64.

Supported application version: 13.3.x (`supported_versions: ">=13.3,<13.4"`). The chart 14.2.0
ships application 13.3.0 as its default. The driver's default version is 13.3.0.

## How to load and deploy

```bash
styrminctl drivers load-version <git-url> <branch> --driver-directory grafana-driver
styrminctl deployments create grafana 13.3.0 <environment> --cluster <cluster>
```

The deployment's URL is `<external_url_scheme>://<primary_fqdn>`. The driver sets Grafana's
`root_url` to that URL.

## Credentials

- Admin user: `admin`.
- Admin password: `styrmin-grafana`, set in `values.j2.yml` (`adminPassword`). The chart stores it
  in the Secret `styrmin-grafana`.

To change the password, edit `adminPassword` in `values.j2.yml`, load a new driver version, and
deploy it. The driver has no parameter for the password, so every deployment of this version
starts with the same default. Not verified: whether a changed `adminPassword` reaches a running
Grafana. Grafana's own documentation says the admin password is set when the admin user is first
created, so after the first start the password may also need changing in Grafana (Administration,
Users).

## Parameters

None.

## Decisions and why

- Persistence on, 10 GiB, Recreate strategy. Grafana stores its state in SQLite under
  `/var/lib/grafana`. Without a volume, the chart uses an emptyDir and every pod restart deletes
  the dashboards and users. The Recreate strategy stops the old pod before the new one starts,
  because a ReadWriteOnce volume cannot be mounted by two pods at once. Size 10 GiB is the chart's
  default; it is not sized from a measured workload.
- One pod (`singleton: true`). Two pods would each write their own SQLite file.
- No internet egress. The chart's default enables Grafana's update check against grafana.com. The
  driver sets `analytics.check_for_updates: false` and allows only DNS and the deployment's own
  pods. Plugin installs from the catalog reach the internet, so they are expected to fail in this
  driver (not verified). If users need the catalog, add an ipBlock rule to `extraObjects` in
  `values.j2.yml`.
- Test pod off (`testFramework.enabled: false`). The chart's helm test pod is not part of a running
  deployment.
- Admin password fixed rather than generated, because Styrmin requires fixed credentials.
- Backup as a disk snapshot of the volume, not a database dump: the driver adds no dump step. A
  snapshot taken while Grafana writes to SQLite could capture a half-written file. Not verified:
  whether a Velero restore of this volume opens Grafana without errors.
- The `search` emptyDir (the bleve index for unified search) is not backed up. Grafana rebuilds it
  on start.
- The chart's ClusterRole has no rules, so it grants nothing. The driver keeps the chart's RBAC
  defaults.
- Grafana's default `grafana.ini` settings are kept, except the ones above.
- Ingress uses the chart's single path `/` with `Prefix`.

## Checked

`check_driver.py /work/grafana-driver --check-images`: 0 FAIL, 1 WARN, no TODO left.

- Spec follows the driver schema. Values render to valid YAML.
- `helm template` renders 10 resources (NetworkPolicy, ServiceAccount, Secret, ConfigMap, PVC,
  ClusterRole, ClusterRoleBinding, Service, Deployment, Ingress).
- Component `server` maps to Deployment `styrmin-grafana`. Service `grafana` is served on its
  primary FQDN. DNS is reachable from every pod. The backup is kind `disk`.
- Image tags exist (see above).
- The rendered ConfigMap contains `root_url = https://<primary_fqdn>` and `check_for_updates = false`.
- WARN `grafana.ini.server.root_url is not a value of the chart`: the check compares keys with the
  chart's `values.yaml`. Grafana's `grafana.ini` is a free-form map, and the chart's `values.yaml`
  only mentions `root_url` in a comment. The rendered ConfigMap has the value, so it takes effect.
  The warning is expected and left in place.
- WARN `component 'server' has a persistent volume but no backup block`: fixed by adding the
  backup block.

## Not verified

- Not deployed on a Styrmin cluster or a Kubernetes cluster. The checks only render the chart and
  query the registries.
- Not verified: that Grafana starts and answers on its URL, that the Ingress works with the
  environment's ingress class, that the PVC binds on the storage class, and that the readiness
  probe passes within 270 seconds.
- Not verified: the Velero restore of the data volume (see the backup decision above).
- Not verified: that the chart at `grafana-community` is the chart the Grafana project
  recommends. Artifact Hub lists it as a verified publisher with the Grafana project's source
  link; the source of the chart itself was not checked.
- Not verified: the Grafana 13.3 release notes for changes that affect this configuration.
