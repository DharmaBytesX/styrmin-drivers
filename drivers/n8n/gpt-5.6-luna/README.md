# n8n driver for Styrmin

This driver deploys n8n 2.44.x from the official n8n Helm chart. It runs the
single-main standalone mode with a 10 GiB persistent volume for n8n's SQLite
data and exposes the editor and webhook endpoint through the external `n8n`
HTTP service.

| Component | Workload | Backup |
|---|---|---|
| `main` | Deployment/styrmin-n8n-main | disk |

- Chart: `oci://ghcr.io/n8n-io/n8n-helm-chart/n8n` 1.17.0 (application 2.44.5).
- Image: `docker.n8n.io/n8nio/n8n:{{ app.version }}`.
- Credentials: n8n has no default username/password in this chart; the first
  browser visit uses n8n's owner-account setup. The fixed
  `N8N_ENCRYPTION_KEY` is a chart-required application secret, not a login.
  Replace it in `values.j2.yml` before production deployment and preserve it
  for the lifetime of the instance so stored credentials remain decryptable.
- Parameters: none.
- Decisions:
  - Standalone mode (`queueMode.enabled: false`) was selected because the
    official chart's queue mode requires externally managed PostgreSQL and
    Redis, while the chart supports standalone SQLite with persistence. This
    keeps the driver self-contained; scale-out queue mode is not provided.
    `database.type: sqlite`, `database.useExternal: false`, and
    `redis.enabled: false` are set explicitly because the chart otherwise
    retains its queue-mode database secret reference in the rendered pod.
  - Persistence is enabled with the Styrmin standard storage class and a 10
    GiB volume because standalone n8n stores its SQLite database under its data
    volume.
  - RBAC creation and service-account token mounting are disabled because n8n
    does not need Kubernetes API access. The chart's default Role and
    RoleBinding would otherwise be unnecessary.
  - Egress includes DNS, same-namespace pods, and public IPv4 addresses because
    workflow automation commonly calls external APIs. Private and carrier-grade
    private ranges are excluded from the public rule.
  - Both service FQDNs and the configured external URL scheme are passed to
    n8n so editor links and webhook URLs match the Styrmin ingress.
- Checked: `check_driver.py --check-images` passes with 0 FAIL. Chart rendering
  produced the expected main Deployment, Service, Ingress, persistent volume
  claim, Secret, and egress NetworkPolicy. The image check reported one WARN
  because Docker's registry endpoint returned HTTP 429 rate limiting; the image
  repository and `2.44.5` tag come directly from the official chart and were
  not changed based on that transient response.
- Not verified: the driver was not loaded into Styrmin or deployed to a
  Kubernetes cluster. TLS behavior, storage provisioning, and first-run owner
  setup were not exercised.
