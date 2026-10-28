# gatus driver for Styrmin

Deploys Gatus 5.34.0, the automated service health dashboard from [TwiN/gatus](https://github.com/TwiN/gatus),
as one Deployment with a Service and an external Ingress. Gatus checks the endpoints in its
configuration and shows their status on a web page. Its results are kept in memory, so a restart clears
the history.

| Component | Workload | Backup |
|---|---|---|
| `server` | Deployment/styrmin-gatus (one container, `gatus`) | none |

- Chart: `oci://ghcr.io/bjw-s-labs/helm/app-template` 5.2.1 (bjw-s app-template, not the Gatus chart; see Decisions).
- Image: `twinproduction/gatus:v5.34.0`, from `default_version` 5.34.0 through `v{{ app.version }}`.
- Service: `gatus`, external, port 8080. Users open the `primary_fqdn` or `human_readable_fqdn` host
  over the scheme and port that Styrmin sets for the instance.
- Credentials: none. This driver sets no login for the dashboard, and it creates no secret.
- Parameters: none.

## Load and deploy

```bash
styrminctl drivers load-version <git-url> <branch> --driver-directory gatus-driver
styrminctl deployments create gatus 5.34.0 <environment> --cluster <cluster>
```

## Decisions

- **app-template instead of the Gatus chart.** The Gatus chart (`https://twin.github.io/helm-charts/gatus`,
  version 1.5.0, published by the Gatus author's chart repository) renders its NetworkPolicy with ingress
  rules only. Styrmin denies all egress, so the pods could not resolve DNS, and the chart has no value
  that adds an egress rule. Gatus is one container with no database, so the driver follows the
  single-container route in `references/networking.md`, based on `examples/app-template/`.
- **Egress policy.** The driver allows DNS, traffic to the pods of its environment, and traffic to the
  internet except the private ranges (10/8, 172.16/12, 192.168/16, 100.64/10). Gatus checks public sites
  this way. Endpoints inside the cluster outside the environment, or on private addresses, are blocked.
  Widen the policy in `values.j2.yml` if such checks are needed.
- **Configuration.** The driver uses the chart's default configuration as its ConfigMap, with one example
  endpoint (`https://example.org`, status 200 and the expected heading). This is a placeholder: edit the
  `config.yaml` entry in `values.j2.yml` to list the services to monitor, then deploy a new driver version.
  The driver has no parameter for the endpoints.
- **Configuration path.** The container reads `GATUS_CONFIG_PATH=/config/config.yaml`, where the ConfigMap
  is mounted. This is set explicitly; I did not confirm the image's default path.
- **No storage and no backup.** Gatus keeps results in memory when no storage is configured. The driver
  mounts no volume, so there is nothing to back up.
- **Replicas.** One replica, the app-template default. Each Gatus instance keeps its own results, so
  scaling it up gives each copy a separate history.
- **Security context.** The pod runs as user and group 65534, the values the Gatus chart sets by default,
  with a read-only root filesystem and all capabilities dropped.
- **Version range.** `supported_versions` is `>=5.34,<5.35`, the application version of the chart the
  driver was checked against. Gatus v5.37.0 is the newest upstream release at the time of writing. It was
  not checked, so the driver does not claim it.
- **Probes.** Liveness and readiness probes call `/health` on port 8080, as in the chart.

## Checked

`check_driver.py --check-images` on this driver: 0 FAIL, 0 WARN. It rendered the chart with app-template
5.2.1, confirmed the image tag `twinproduction/gatus:v5.34.0` exists for amd64 and arm64, and confirmed
every pod reaches DNS. The `info.icon` URL returns HTTP 200.

## Not verified

- Not deployed on a cluster. The checks render the manifests and check the image; they do not start Gatus.
- Gatus is not started with this configuration, and the endpoints are not checked from inside a cluster.
- The dashboard's Ingress host and TLS behavior are not tested against a real ingress controller.
- Gatus v5.35 to v5.37 are not checked.
