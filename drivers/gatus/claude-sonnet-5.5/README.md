# Gatus driver for Styrmin

Deploys Gatus 5.34.0 (https://github.com/TwiN/gatus), a health dashboard, as one Deployment with
the image `twinproduction/gatus:v<app version>`. The chart is bjw-s app-template 5.2.1
(`oci://ghcr.io/bjw-s-labs/helm/app-template`). The configuration is a ConfigMap mounted at `/config`.

- Component: `server` (container `app`, singleton).
- External service: `gatus`, HTTP, port 8080, on `primary_fqdn` and `human_readable_fqdn`.
- Supported versions: `>=5,<6`; default `5.34.0`. Only 5.34.0 was checked (image tag exists).

## Load and deploy

    styrminctl drivers load-version <git-url> <branch> --driver-directory gatus-driver
    styrminctl deployments create gatus 5.34.0 <environment> --cluster <cluster>

## Credentials and parameters

- Credentials: none. Gatus has no login in its default configuration, and the driver sets none.
- Parameters: none.

## Decisions and reasons

- Chart: the Gatus chart from TwiN (`https://twin.github.io/helm-charts/gatus` 1.5.0) is the only
  chart of the application's own author, but Artifact Hub lists it as "community". It has no value
  for extra objects, and its `networkPolicy` has no egress rules. Under Styrmin the pod could not
  resolve DNS (check_driver.py: FAIL on DNS). Gatus is one container, so I followed the skill's
  route for that case and used app-template.
- Storage: Gatus keeps results in memory (its default). I did not add a volume or SQLite, to keep the
  chart defaults and avoid unchecked settings. Check history is lost when the pod restarts.
- `singleton: true`: with in-memory results, a second replica would show different results.
- Configuration: the example endpoint from the Gatus chart's default values (`https://example.org`).
  Replace the `config.yaml` content in `values.j2.yml` with your own endpoints. The configuration
  is not a parameter because the skill gave no example of a multi-line parameter.
- Egress policy: allows DNS, pods of the same environment, and the internet except private ranges,
  copied from the skill's Uptime Kuma example. Endpoints on private addresses outside the
  environment cannot be monitored until you change the policy.
- Health probes: `/health`, which Gatus serves (not confirmed on a running pod).
- Ingress class: set from `instance.config.ingress_class`, as in the skill's app-template example.
- No backup: nothing is stored on a volume.

## Checked

`check_driver.py --check-images`: 0 FAIL, 0 WARN. Spec, template, `helm template`, components,
service host, DNS egress and the image tag in the registry.

## Not verified

- Not deployed on a cluster. Not verified: that the pod becomes Ready, that `/health` answers, that
  the ConfigMap mount at `/config` is read, and that the dashboard answers on its URL.
- Versions other than 5.34.0.
