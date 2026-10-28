# n8n driver for Styrmin

This driver deploys n8n, the workflow automation tool, as a single pod (standalone mode) that stores its SQLite database and files on one persistent volume. Users reach the editor and webhooks on the deployment's primary and human-readable hosts.

| Component | Workload | Backup |
|---|---|---|
| `main` | Deployment/styrmin-n8n-main | disk (volume `data`) |

- Chart: `oci://ghcr.io/n8n-io/n8n-helm-chart/n8n` 1.17.0 (the official n8n chart, application 2.44.5).
- Image: `docker.n8n.io/n8nio/n8n:{{ app.version }}`, for example `2.44.5`. Supported versions: `>=2.44,<3`; default `2.44.5`.
- Load: `styrminctl drivers load-version <git-url> <branch> --driver-directory <path/in/repo>`
- Deploy: `styrminctl deployments create n8n 2.44.5 <environment> --cluster <cluster>`
- Credentials: n8n has no default user. The first visitor to the URL creates the owner account in the setup screen, so do that right after deployment. The encryption key for stored credentials is the fixed value `styrmin-n8n-encryption-key-0001` in `values.j2.yml` (`secretRefs.env.N8N_ENCRYPTION_KEY`). Change it before the first deployment; changing it later makes saved credentials unreadable.
- Parameters: none.

## Decisions

- Standalone mode with SQLite, not queue mode. Queue mode needs an external PostgreSQL and Redis, and the chart has no subcharts for them. The driver has no verified way to supply them, so it uses the one-pod mode the chart documents. Limit: n8n cannot scale beyond one main pod, and there are no workers.
- `strategy: Recreate`, because the volume is ReadWriteOnce and a rolling update would hit Multi-Attach errors.
- `pdb.enabled: false`, because a disruption budget on one replica blocks node drains.
- `rbac.create: false`. The chart's Role lets n8n read ConfigMaps and Secrets, which n8n does not need, and the check flagged it as a Kubernetes API use that Styrmin's network policies block.
- Egress allows DNS, pods of the same deployment and public internet addresses (private ranges excluded), because workflows call external APIs.
- `N8N_HOST` and `N8N_PROTOCOL` come from `services['n8n'].primary_fqdn` and `external_url_scheme`; the chart derives the webhook URL from them. `N8N_EDITOR_BASE_URL` is set to the same URL. The URL ignores `external_url_port`; this is not verified for non-default ports.
- Supported versions `>=2.44,<3`: only 2.44.5 is the chart's app version; other 2.x versions are not tested with chart 1.17.0.
- Task runners are left off (chart default), so Code nodes run in the main process.

## Checked

`check_driver.py --check-images`: 0 FAIL, 1 WARN. The WARN is an HTTP 429 from docker.n8n.io. I confirmed the tag `n8nio/n8n:2.44.5` exists on Docker Hub (HTTP 200), which `docker.n8n.io` mirrors; I did not confirm it on docker.n8n.io itself.

## Not verified

- Not deployed on a cluster; only the offline checks ran.
- Disk backup and restore of the SQLite volume were not tried.
- Behavior with other 2.x versions, and with a non-default external URL port.
