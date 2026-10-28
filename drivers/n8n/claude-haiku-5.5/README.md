# n8n driver for Styrmin

This driver deploys n8n, the workflow automation tool, for a Styrmin deployment. It runs one n8n
main process that stores its data in an SQLite database on a 10 Gi persistent volume. Users open
the editor at the deployment's URL. The chart's queue mode, which needs an external PostgreSQL
database and Redis, is off.

| Component | Workload | Backup |
|---|---|---|
| `main` | Deployment/styrmin-n8n-main, container `n8n-main`, image `docker.n8n.io/n8nio/n8n:<app version>` | disk snapshot of the `data` volume (`/home/node/.n8n`) |

Service: `n8n`, external HTTP on port 5678, served on the primary host and the human-readable host.
Network: an egress NetworkPolicy allows DNS and outbound traffic to the internet, and blocks the
private ranges 10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16 and 100.64.0.0/10.

- Chart: `oci://ghcr.io/n8n-io/n8n-helm-chart/n8n` 1.17.0 (application 2.44.5), from the n8n-io
  organization's own chart repository. Its Artifact Hub listing is marked official and verified.
- Supported application versions: `>=2.44,<2.45` (the chart 1.17.0 application version, 2.44.5). Only 2.44.5 was checked.
- Credentials: none that users log in with. n8n creates its owner account when the first user
  opens the editor (not verified on a cluster). The one fixed secret is the encryption key
  `styrmin-n8n-encryption-key`, set in `values.j2.yml` under `secretRefs.env`. Styrmin drivers use
  fixed credentials, not generated ones, so the value is the same in every deployment of this
  driver.
- Parameters: none.
- Load and deploy: see the Styrmin README at the repository root:
  `styrminctl drivers load-version <git-url> <branch> --driver-directory n8n-driver`, then
  `styrminctl deployments create n8n <app-version> <environment> --cluster <cluster>`.

## Decisions

- Single main process, no queue mode. The chart's default is queue mode with an external
  PostgreSQL and Redis. This driver deploys neither, so it turns queue mode, the Redis subchart
  and the worker off, and uses `database.type: sqlite`. The chart renders the one main
  Deployment and no worker with these values (checked with `helm template`).
- Singleton. SQLite allows one writer, so the `main` component is `singleton: true`.
- Persistence: 10 Gi on the environment's standard storage class, mounted at `/home/node/.n8n`.
  The size is the chart's default and not a sizing result.
- Backup: `kind: disk` on the `data` volume. The volume holds the SQLite database. The backup
  copies the volume as a snapshot. Whether a snapshot taken while n8n writes restores a consistent
  database is not known: it was not tested.
- Chart RBAC off (`rbac.create: false`). The chart's Role lets pods read ConfigMaps and Secrets
  through the Kubernetes API. The single-main SQLite setup does not use it, and Styrmin's network
  policies block that traffic. The service account is still created.
- No PodDisruptionBudget (`pdb.enabled: false`). With one replica, a PDB would block node drains.
- Host settings: the chart's default host `n8n.example.com` is replaced by the primary host, and the
  editor URL is `{{ instance.config.external_url_scheme }}://<primary host>`, so n8n builds its
  links from the deployment's URL.
- Internet egress is allowed. n8n workflows call external services, so the driver keeps the
  outbound rule for the internet. Remove the `ipBlock` rule in `values.j2.yml` to block it.
- Sidecars and init containers: none. The render with these values has one container, `n8n-main`,
  and no init containers. The chart's preStop lifecycle hooks are kept.

## Checked

`check_driver.py --check-images` reports 0 FAIL and 0 WARN:

- Spec follows the driver schema; values render to valid YAML; `helm template` renders 9
  resources with the chart 1.17.0.
- Component `main` maps to Deployment/styrmin-n8n-main; service `n8n` is served on its primary
  host.
- Every pod can reach DNS.
- Backup kind `disk` is set.
- Image `docker.n8n.io/n8nio/n8n:2.44.5` exists for linux/amd64 and linux/arm64.

An earlier run reported a WARN for the chart's Role (the Kubernetes API) and a WARN for the missing
backup; both are fixed as described in the decisions.

## Not verified

- Not deployed on a Kubernetes cluster and not started. The checks render the chart offline.
- Not verified that n8n starts with SQLite on the volume, creates its owner account, or runs a
  workflow.
- Not verified that the disk snapshot restores a working SQLite database.
- Not known: which destinations real workflows need. The internet rule allows all of them except
  the private ranges.
- Not verified: application versions other than 2.44.5. The range `>=2.44,<2.45` was not tested
  beyond that version.
