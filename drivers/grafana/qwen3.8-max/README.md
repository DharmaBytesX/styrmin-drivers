# grafana driver for Styrmin

This driver deploys [Grafana](https://grafana.com), the open-source platform for querying,
visualizing and alerting on metrics and logs, from the official
[`grafana/helm-charts`](https://github.com/grafana/helm-charts) chart. Grafana runs as a single
StatefulSet pod with its data (an embedded SQLite database, dashboards and installed plugins) on a
persistent volume, and is exposed on one external URL.

| Component | Workload | Container | Backup |
|---|---|---|---|
| `server` | StatefulSet/styrmin-grafana | `grafana` | disk (volume `storage`) |

- **Chart:** `https://grafana.github.io/helm-charts/grafana` version `10.5.15` (application
  `12.3.1`). This is the official chart published by Grafana Labs. `find_chart.py` lists a
  `grafana-community` mirror first, but that location returned HTTP 404 on pull, so the driver
  points at the canonical `grafana.github.io/helm-charts` repository that Artifact Hub backs with
  the same chart.
- **Images:** `docker.io/grafana/grafana:{{ app.version }}` (e.g. `12.3.1`) and the chart's
  `initChownData` helper `docker.io/library/busybox:1.31.1`. Both confirmed present for
  linux/amd64 and linux/arm64 by `check_driver.py --check-images`.

## Load and deploy

```bash
styrminctl drivers load-version <git-url> <branch> --driver-directory grafana-driver
styrminctl deployments create grafana 12.3.1 <environment> --cluster <cluster>
```

Open `{{ external_url_scheme }}://<primary_fqdn>` (the URL Styrmin shows for the deployment) and
sign in.

## Credentials

Grafana ships with a single administrator account. Both values are **fixed** (never randomly
generated), so a restored or cloned deployment keeps working with its data.

| Setting | Default | Parameter |
|---|---|---|
| Admin user | `admin` | `admin_user` |
| Admin password | `admin-password` | `admin_password` |

Change them at deploy time by overriding the parameters, or later from the Grafana UI. The chart
writes them to the `styrmin-grafana` Secret (`admin-user` / `admin-password`) and injects them as
`GF_SECURITY_ADMIN_USER` / `GF_SECURITY_ADMIN_PASSWORD`.

## Parameters

| Name | Kind | Default | Purpose |
|---|---|---|---|
| `admin_user` | string | `admin` | Administrator username |
| `admin_password` | string | `admin-password` | Administrator password |
| `storage_size` | integer | `10` | Size of the data volume in GiB |

## Decisions and reasons

- **StatefulSet instead of Deployment.** Grafana is a single-writer data store (its SQLite database
  lives on the volume), so it must run exactly one pod. `useStatefulSet: true` makes the chart
  render a StatefulSet with a stable `volumeClaimTemplates` entry named `storage`, which the
  disk-backup strategy snapshots. The component is also marked `singleton: true`. (The chart's
  `persistence.type: pvc` + Deployment path names the volume `storage` too but does not give a
  stable identity across reschedules.)
- **Backup `kind: disk` on volume `storage`.** There is no dump-based strategy for Grafana's
  embedded SQLite; snapshotting the PVC (dashboards, plugins, users, settings) is the correct
  choice. `storage` is the pod-template volume name the chart mounts at `/var/lib/grafana`.
- **Storage class from context, not hard-coded.** `persistence.storageClassName` =
  `{{ instance.config.standard_storage.storage_class }}`; size from the `storage_size` parameter.
- **`rbac.create: false`, ServiceAccount kept.** No subchart, sidecar or renderer is enabled, and
  nothing calls the Kubernetes API, so the chart's Role/ClusterRole are unnecessary. Removing them
  avoids cluster-scoped objects a single deployment should not need. The ServiceAccount is retained
  because the pod spec references it.
- **`initChownData` kept.** This init container only runs `chown` on the volume (no Kubernetes API
  call, so it is not blocked by Styrmin's policies). It makes the PVC writable by Grafana's
  non-root user `472`.
- **`testFramework.enabled: false`.** The bundled bats Helm-test Job is not part of a Styrmin
  deployment.
- **Image tag is the bare version.** The registry tags have no `v` prefix (`grafana/grafana:12.3.1`),
  so `image.tag: "{{ app.version }}"`. Supported range `>=12.3,<12.4` matches the versions the
  pinned chart 10.5.15 ships; `12.3.0` and `12.3.1` both render and their image tags exist.
- **Ingress with both hosts, no class.** `ingress.enabled: true` with `primary_fqdn` and
  `human_readable_fqdn` as `ingress.hosts` (the chart takes a list of host strings and builds
  `path: /` `pathType: Prefix` for each). Styrmin patches the ingress class, so it is not set here.
- **`grafana.ini` server.domain / root_url / analytics.** `server.domain` and `server.root_url` are
  set from the context so Grafana builds correct absolute URLs behind the ingress;
  `analytics.check_for_updates` and `analytics.reporting_enabled` are `false` so a deployment
  becomes Ready without any internet access.
- **Egress NetworkPolicy without an internet rule.** Styrmin denies all egress in the namespace, so
  the driver ships a policy (via the chart's `extraObjects`) allowing DNS and the pods of the same
  deployment. Grafana runs fully offline with an empty data-source set, so no `0.0.0.0/0` rule is
  included. **To install plugins from grafana.com or reach data sources outside the cluster, add
  the internet `ipBlock` rule** (see the comment in `values.j2.yml`).

## Checked

`check_driver.py /work/grafana-driver --check-images`: **0 FAIL, 2 WARN**.

The two WARNs are false positives:

```
WARN [values] grafana.ini.analytics.reporting_enabled is not a value of the chart
WARN [values] grafana.ini.server.root_url is not a value of the chart
```

`grafana.ini` is a free-form map that the chart converts to ini format (see
`templates/configmap.yaml` and `_config.tpl`), so the check script cannot match arbitrary
sub-sections against `values.yaml`. Both keys render correctly into the ConfigMap
(`root_url` and `reporting_enabled` appear in the rendered `grafana.ini` in
`/tmp/check/manifests.yaml`), and `server.root_url` is referenced by the chart's own default
`grafana.ini`. They are kept intentionally.

Everything else passes: spec schema, template render, `helm template` (8 resources), component
`server` -> StatefulSet/styrmin-grafana, external service on its `primary_fqdn`, DNS reachable,
disk backup, and both images exist for linux/amd64 and linux/arm64.

## Not verified

- **Not deployed on a cluster.** The driver was validated offline only (schema, render,
  `helm template`, image-tag existence). Pod readiness timing, actual disk backups via Velero, and
  reaching the UI through a real ingress were not exercised on a live Styrmin/kind cluster.
- The `storage_size` parameter is passed straight to `persistence.size`; Grafana does not resize an
  existing volume automatically, so grow it before the PVC is created or expand the PVC manually.
