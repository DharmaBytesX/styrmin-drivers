# Grafana driver for Styrmin

Deploys [Grafana](https://github.com/grafana/grafana) (the observability dashboards) with the
official community Helm chart
[`grafana-community/grafana`](https://github.com/grafana-community/helm-charts) **14.2.0**
(application **13.3.0**), on a persistent volume, behind Styrmin's Ingress, with the egress
NetworkPolicy that Styrmin needs.

| Component | Workload | Image | Backup |
|---|---|---|---|
| `server` | `Deployment/styrmin-grafana` (container `grafana`) | `docker.io/grafana/grafana:13.3.0-distroless` | `disk` of the pod volume `storage` |

Objects the chart renders in the deployment namespace: the Deployment (`styrmin-grafana`), the
Service `styrmin-grafana` (port 80 -> container port 3000), the Ingress `styrmin-grafana`, the
ConfigMap `styrmin-grafana` (the rendered `grafana.ini`), the Secret `styrmin-grafana` (the admin
credentials), the ServiceAccount `styrmin-grafana`, the PersistentVolumeClaim `styrmin-grafana`
and, from the driver's values template, the NetworkPolicy `styrmin-grafana-egress`.

The users open the URL Styrmin gives the external service `grafana`:

```
{{ instance.config.external_url_scheme }}://<primary_fqdn>[:<external_url_port>]
```

For example `https://grafana-<deployment id>.<fqdn suffix>`; the second host
(`grafana-<environment>.<fqdn suffix>`) is on the same Ingress.

## Load and deploy

```bash
styrminctl drivers load-version <git-url> <branch> --driver-directory <path/in/repo>
styrminctl deployments create grafana 13.3.0 <environment> --cluster <cluster>
```

The highest `spec.version` wins when several loaded driver versions match; pass
`--driver-version-id <uuid>` to pin one. The parameters below can be set on the deployment
(`spec.parameters`).

## Credentials

| User | Password | Where it comes from |
|---|---|---|
| `admin` | `grafana-admin` | the `admin_password` parameter (default `grafana-admin`) |

The password is fixed, not generated: the chart generates a random one (`randAlphaNum 40`) when
`adminPassword` is empty, and a restored or cloned deployment would then not match its data.
**Change it** by setting the `admin_password` parameter when the deployment is created; the value
is stored in the chart's Secret and in the `GF_SECURITY_ADMIN_PASSWORD` environment variable of
the container.

## Parameters

| Name | Kind | Default | Meaning |
|---|---|---|---|
| `admin_password` | string | `grafana-admin` | Password of the `admin` user. Change it. |
| `storage_size` | string | `10Gi` | Size of the PersistentVolumeClaim that holds the Grafana data (SQLite database, dashboards, users, plugins). |

## Decisions, and why

1. **Chart: `grafana-community/grafana` 14.2.0.** It is the chart of Grafana's own community
   organisation, listed as a verified publisher on Artifact Hub, and its `appVersion` is 13.3.0.
   The Bitnami chart was not used: Bitnami has moved its versioned images to
   `docker.io/bitnamilegacy`, which would have pinned Grafana to an old patch release.
2. **Image tag `{{ app.version }}-distroless`.** The chart's default tag is the app version with
   the `-distroless` suffix, and the tags of the 13.3 line carry no `v` prefix
   (`13.3.0-distroless`, not `v13.3.0`). `check_driver.py --check-images` confirms that
   `docker.io/grafana/grafana:13.3.0-distroless` exists for `linux/amd64` and `linux/arm64`.
3. **`supported_versions: ">=13.3,<13.4"`, `default_version: "13.3.0"`.** The pinned chart ships
   appVersion 13.3.0, and 13.3.0 is the only published release of the 13.3 line. The range can be
   widened when a newer chart (and its app version) is pinned.
4. **Persistence on, with the storage class of the environment.**
   `persistence.storageClassName` is `{{ instance.config.standard_storage.storage_class }}`.
   Without it Grafana's SQLite database, dashboards, users and installed plugins live in an
   `emptyDir` and are lost on every restart.
5. **The component is a `singleton` with a `disk` backup.** Grafana's SQLite database has a single
   writer on a `ReadWriteOnce` volume, so the deployment must never run two pods. The backup
   snapshots the pod volume `storage` (the name under `volumes:` in the pod spec, not the name of
   the PVC).
6. **`grafana.ini.server.root_url` is set explicitly.** The chart's default `root_url` is
   `%(protocol)s://%(domain)s:%(http_port)s/`, that is `http://<primary_fqdn>:3000/` - the
   container port, which the Ingress does not serve. The driver sets the public URL
   (`external_url_scheme` + `primary_fqdn` + `external_url_port`, the port only when it is not
   80/443), so that links, redirects, alert contact points and e-mails use the address the users
   open. The chart's `domain` (the first Ingress host) stays the primary host, and
   `enforce_domain` is left at Grafana's default (`false`), so a request that arrives on the
   second host is served, not rejected with "Invalid Host".
7. **Egress to the internet is allowed** (without the private ranges, as Styrmin recommends).
   Grafana needs it to install plugins from grafana.com, to check for updates, to reach its
   public APIs and to proxy requests to datasources that live outside the cluster. Data sources
   that run in other namespaces of the same environment are reachable through Styrmin's own
   `allow-egress-to-environment` policy; the pods of the deployment itself are allowed by the
   `podSelector: {}` rule.
8. **`rbac.create: false` and `automountServiceAccountToken: false`.** The driver deploys no
   sidecar and no dashboard provider, so the chart's ClusterRole would carry no rules at all;
   not creating it keeps the install inside the deployment's namespace and gives the pod no
   Kubernetes API token.
9. **A `startupProbe` with 30 x 10 s.** On the first start Grafana migrates its SQLite database.
   The chart's liveness probe starts after 60 s and would restart the container during a slow
   migration; the startup probe gives it up to 300 s before the liveness probe takes over. The
   first start is therefore slower than the later ones.
10. **`testFramework.enabled: false`.** The chart's test pod is not part of the deployment.
11. **No `actions.py`.** Helm expresses everything this driver does: install, upgrade and
    uninstall are Styrmin's own steps.
12. **No resource requests or limits.** They are left unset so that the pod starts on any
    cluster; the component accepts `cpu` and `memory` keys if a future driver version wants to
    request them.
13. **No pre-installed plugins, dashboards or data sources.** They are configured in the Grafana
    UI, which is what an observability deployment usually needs; the egress policy of point 7
    allows the plugin catalogue to work.

## What the check script reported

```
$ python3 <skill>/scripts/check_driver.py /work/grafana-driver --check-images
OK   [spec] driver.styrmin.yml follows the driver schema
OK   [template] values.j2.yml renders to valid YAML
WARN [values] grafana.ini.server.root_url is not a value of the chart: Helm ignores it without an error
OK   [chart] helm template rendered 8 resources from https://grafana-community.github.io/helm-charts/grafana 14.2.0
OK   [components] component 'server' -> Deployment/styrmin-grafana
OK   [services] external service 'grafana' is served on its primary_fqdn
OK   [network] every pod can reach DNS
OK   [backup] component 'server' backup kind disk
OK   [images] docker.io/grafana/grafana:13.3.0-distroless: exists for linux/amd64 and linux/arm64
OK   [images] docker.io/library/busybox:1.38.0: exists for linux/amd64 and linux/arm64

0 FAIL, 1 WARN.
```

**The one WARN is a false positive.** The check compares every key of the rendered values with the
keys of the chart's `values.yaml`; `grafana.ini` is an open map there (the chart's default only
lists `domain` under `[server]`), and the chart's template writes every key of a `grafana.ini`
section into the `grafana.ini` file. The rendered ConfigMap of the check proves that the value is
used, and not ignored:

```
[server]
domain = grafana-<deployment id>.apps.example.com
root_url = https://grafana-<deployment id>.apps.example.com
```

The alternative (no `root_url`) would give Grafana the container's port in its public URL, see
decision 6.

## Not verified

- The driver was **not deployed on a cluster**: there is no Styrmin installation and no
  Kubernetes cluster in this environment. Only the offline checks of
  `check_driver.py` (spec, template, `helm template`, images in the registry) were run.
- The login, the datasources and the alerting of Grafana were not exercised at runtime.
- Only the version 13.3.0 was rendered; the range `>=13.3,<13.4` currently contains that single
  published release.
