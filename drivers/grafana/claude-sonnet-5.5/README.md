# grafana driver for Styrmin

This driver deploys Grafana (observability dashboards) as one Deployment with a SQLite database on a persistent volume. Users open it at the deployment's URL and sign in as `admin`.

| Component | Workload | Backup |
|---|---|---|
| `server` | Deployment/styrmin-grafana | disk, volume `storage` (mounted at `/var/lib/grafana`) |

- Chart: `https://grafana-community.github.io/helm-charts/grafana` 14.2.0 (application 13.3.0).
- Image: `docker.io/grafana/grafana:{{ app.version }}` (for example `13.3.0`). The tag exists for linux/amd64 and linux/arm64 (checked by `check_driver.py --check-images`).
- External service: `grafana`, HTTP, container port 3000, on `primary_fqdn` and `human_readable_fqdn`.
- Supported versions: `>=13.3,<13.4`. Source for the range: the chart's `appVersion` (13.3.0). Other Grafana versions were not tested with this chart version.

## Load and deploy

```bash
styrminctl drivers load-version <git-url> <branch> --driver-directory grafana-driver
styrminctl deployments create grafana 13.3.0 <environment> --cluster <cluster>
```

## Credentials

- User `admin`, password `styrmin-grafana-admin`, set in `values.j2.yml` (`adminUser`, `adminPassword`).
- The password is public in the driver. Change it in Grafana after the first sign-in (Profile, Change password), or edit `adminPassword` in a copy of the driver. Whether a later change of `adminPassword` replaces a password changed in the UI was not verified.

## Parameters

None.

## Decisions and reasons

- Chart choice: the `grafana-community` chart was listed first on Artifact Hub, as a verified publisher, with the newest application version (13.3.0). The Bitnami chart is on 12.1.8 and ships an older Grafana. The Grafana project's own chart repository was not in the search results, so it was not compared.
- Image tag: `{{ app.version }}` without the chart's `-distroless` suffix, following the scaffold and the standard image tag. The chart default is the `-distroless` image. Distroless was not tested.
- One component, `singleton: true`, with the `Recreate` update strategy: SQLite on a ReadWriteOnce volume cannot be shared by two pods, including during a rolling update.
- Persistence is on (10Gi, storage class from the context), so dashboards, users and data sources survive restarts. Disk backup uses the chart's volume `storage`.
- `initChownData` is off: it runs a root `busybox` init container, and the chart's pod security context already sets `fsGroup`. `testFramework` is off: its test pod is not needed.
- `grafana.ini.server.root_url` is set from `external_url_scheme`, the primary host and `external_url_port` (the port is added only when it is not 80 or 443), so links and redirects use the public URL. The chart sets `server.domain` from the first ingress host.
- `check_for_updates` is off, because egress to the internet is denied.
- Egress NetworkPolicy: DNS and pods of the same deployment only. No internet access, so installing plugins from grafana.com and queries to data sources outside the deployment fail. A user who needs these must add an egress rule for the data source's address to `values.j2.yml`. The driver cannot know those addresses.
- Sidecars (dashboards and data source discovery), the image renderer and subchart-style extras are off, as in the chart defaults.
- The chart creates a ClusterRole and ClusterRoleBinding (`rbac.create` is true by default). They are left as is because the sidecars that use them are off; the check script reported no problem. Whether the cluster allows a deployment to create them was not verified.

## Checked

`check_driver.py . --check-images`: 0 FAIL, 1 WARN.

- WARN `grafana.ini.server.root_url is not a value of the chart`: not a defect. `grafana.ini` is a free-form map that the chart writes to `grafana.ini`; its `root_url` entry is only a comment in the chart's `values.yaml`. The rendered ConfigMap in `/tmp/check/manifests.yaml` contains `root_url = https://<host>/`.

## Not verified

- Not deployed on a cluster. Pod readiness within 270 s, the ingress, sign-in and the disk backup were not run.
- Login with `admin` and the fixed password was not tested.
- Versions other than 13.3.x.
