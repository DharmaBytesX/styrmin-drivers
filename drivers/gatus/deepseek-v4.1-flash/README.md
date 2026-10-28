# gatus driver for Styrmin

Deploys [Gatus](https://github.com/TwiN/gatus), the automated service health dashboard, as one
container. The driver renders the [bjw-s app-template](https://github.com/bjw-s-labs/helm-charts)
chart (5.2.1): a Deployment `styrmin-gatus` running `twinproduction/gatus:v<app.version>`, a
ClusterIP Service on port 8080, a ConfigMap holding `config.yaml` mounted read-only at `/config`,
an Ingress on the two Styrmin hosts, and an egress NetworkPolicy. Gatus stores check results in
memory, so there is no volume and no backup.

| Component | Workload | Backup |
|---|---|---|
| `server` | Deployment/styrmin-gatus | none |

- Chart: `oci://ghcr.io/bjw-s-labs/helm/app-template` 5.2.1, running the Gatus image
  `twinproduction/gatus` (application 5.34.0 by default; supported range `>=5.34,<5.35`).
- Credentials: none. Gatus has no authentication by default, so anyone who can open the URL sees
  the dashboard and can silence or inspect results. Add basic auth, OIDC or SSO through the
  `config` parameter (Gatus `security:` section); the secret is then part of the parameter value
  in the deployment configuration, not a Kubernetes Secret created by this driver. There are no
  default users or passwords to change.
- Parameters:
  - `config` (string, default shown below): the complete Gatus configuration written to
    `/config/config.yaml`. Replace the default self-check with your endpoints. It is inserted
    verbatim, so keep the YAML indentation.
- Decisions:
  - **Chart.** The official Gatus chart (`https://twin.github.io/helm-charts/gatus` 1.5.0, by
    TwiN, the application's author) cannot run under Styrmin: it has no `extraDeploy`/`extraObjects`
    value and its `networkPolicy` values only create ingress rules, so its pod could not resolve
    DNS under Styrmin's default-deny egress. Gatus is a single container, so per the skill this
    driver uses app-template, which renders the same Deployment, Service and ConfigMap and can
    add an egress NetworkPolicy. The geek-cookbook chart (1.1.2, app 3.8.0) was rejected as stale.
  - **Image.** The official image `twinproduction/gatus`, tag `v{{ app.version }}`, the same image
    and tag format as the Gatus chart. `--check-images` confirms the default tag exists for
    linux/amd64 and linux/arm64.
  - **Configuration.** The driver ships a working default instead of the upstream chart's
    `https://example.org` probe: Gatus checks its own `/health`, so a fresh deployment is green.
    The whole configuration is a parameter because endpoints are the reason to run Gatus, and
    Helm values in a driver are fixed at load time.
  - **Storage.** In-memory, matching the Gatus chart default (`persistence.enabled: false`):
    check history is lost on restart. To keep it, add a PVC and Gatus `storage: {type: file,
    path: /data}`; this driver does not, so it needs no singleton and no backup.
  - **Ingress.** Both Styrmin hosts (`primary_fqdn`, `human_readable_fqdn`) with path `/`
    (Prefix); the class comes from `instance.config.ingress_class`, as in the app-template
    example. TLS is left to Styrmin/the environment.
  - **Egress.** Allows DNS, the pods of the environment (where monitored services usually live)
    and the internet minus private ranges. This follows the skill's rule not to allow private
    ranges (the cluster, including Styrmin's server, lives there). Consequence: Gatus cannot
    probe a bare private IP of the cluster; put the target behind a Service in the environment,
    or reach it through its public URL.
  - **Security context.** Runs as uid/gid 65534 with a read-only root filesystem, as the Gatus
    chart does. The ConfigMap is mounted read-only, so no writable path is needed with memory
    storage.
- Checked: `check_driver.py --check-images` reports **0 FAIL, 0 WARN**: the spec validates, the
  template renders, `helm template` produces 6 resources, component `server` selects
  Deployment/styrmin-gatus, the external service host answers, every pod resolves DNS, and the
  image tag exists. Only app version 5.34.0 was rendered; the range is set for patch/minor
  updates within 5.34.
- Not verified: not deployed on a Kubernetes cluster, so the Ingress, the runtime behaviour of
  the probes and the actual outbound checks were not exercised. Gatus needs real network egress
  from the cluster (not only a NetworkPolicy) to reach public endpoints. UI support for entering
  a multi-line `config` value was not verified.

## Load and deploy

```bash
styrminctl drivers load-version <git-url> <branch> --driver-directory <path/in/repo>
styrminctl deployments create gatus 5.34.0 <environment> --cluster <cluster>
```

Default configuration:

```yaml
storage:
  type: memory
endpoints:
  - name: gatus
    group: internal
    url: http://localhost:8080/health
    interval: 60s
    conditions:
      - "[STATUS] == 200"
```
