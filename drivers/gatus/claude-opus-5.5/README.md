# Gatus driver for Styrmin

Deploys [Gatus](https://github.com/TwiN/gatus), a health dashboard that checks endpoints at an
interval and shows their status and history. One Deployment runs the image
`twinproduction/gatus:v<version>`; its history is in SQLite on a 1 GiB volume.

| Component | Workload | Backup |
|---|---|---|
| `server` | Deployment `styrmin-gatus`, container `app`, port 8080, singleton | `disk`, volume `data` (`/data`, the SQLite file) |

- Chart: bjw-s app-template 5.2.1 (`oci://ghcr.io/bjw-s-labs/helm/app-template`), not Gatus's
  own chart: see Decisions.
- Service `gatus` (external): the Ingress serves `primary_fqdn` and `human_readable_fqdn` on `/`.
- Versions: `>=5.0,<6.0`, default `5.34.0`.
- Credentials: none. Gatus has no login by default: anyone who reaches the URL sees the
  dashboard. Gatus can add basic or OIDC authentication in its configuration (`security`).
- Parameters: none.
- Configuration: `configMaps.config` in `values.j2.yml` holds `config.yaml`. It keeps the example
  endpoint of Gatus's chart (`https://example.org`, every 60 s). Replace `endpoints` with what you
  want to monitor, then load the driver as a new version.

## Load and deploy

```bash
styrminctl drivers load-version <git-url> <branch> --driver-directory <path>/gatus-driver
styrminctl deployments create gatus 5.34.0 <environment> --cluster <cluster>
```

## Decisions

- **app-template instead of Gatus's chart.** Gatus's chart (`https://twin.github.io/helm-charts`,
  gatus 1.5.0, named in the Gatus README) has no value for extra objects, and its `networkPolicy`
  only allows incoming traffic. Styrmin denies all egress in the deployment namespace, so with
  that chart the pod could not resolve DNS nor reach any endpoint. app-template renders the same
  pod as Gatus's chart (image, port 8080, `/health` probes, read-only root file system, user and
  group 65534, `fsGroup` 65534, configuration mounted at `/config`) and adds the NetworkPolicy.
- **Egress:** DNS, the pods of the environment (Gatus can check the other applications), and the
  internet without private ranges (Gatus checks public URLs). Without the internet rule, the
  example endpoint fails.
- **SQLite on a volume** (`storage.type: sqlite`), so the history survives a restart; Gatus's
  chart keeps it in memory by default. This makes the pod a singleton with a `Recreate` strategy.
- **Versions `>=5.0,<6.0`:** the image tags `v5.0.0` and `v5.34.0` exist for amd64 and arm64.
  Only 5.34.0 was deployed.

## Checked

- `check_driver.py --check-images`, with `--app-version 5.34.0` and `5.0.0`: 0 FAIL, 0 WARN.

## Not verified

- Versions other than 5.34.0 on a cluster; backups and restores; upgrades.
