# vault driver for Styrmin

This driver deploys HashiCorp Vault, a secrets manager, as one standalone server on Kubernetes. The server
runs on a single StatefulSet with a persistent data volume (audit storage is off), and an external
Service exposes its API on port 8200 through two Ingress hosts. A new server starts sealed. An
operator runs `vault operator init` and unseals it before clients can read or write secrets.

| Component | Workload | Backup |
|---|---|---|
| `server` | StatefulSet/styrmin-vault | disk (volume `data`) |

- Chart: `https://helm.releases.hashicorp.com/vault` 0.34.1 (application 2.0.4).
- Image: `hashicorp/vault`, tag set from the application version (`{{ app.version }}`).
- Supported application versions: `>=2.0,<2.1`. The check script rendered and checked 2.0.0, 2.0.1 and 2.0.4.
- Credentials: none set by the driver. Vault creates its root token and unseal keys when an operator runs
  `vault operator init`. The chart has no value that sets them, and the driver does not store them.
  Dev mode is off, so no fixed root token exists.
- Parameters: none.

## Load and deploy

```bash
styrminctl drivers load-version <git-url> <branch> --driver-directory vault-driver
styrminctl deployments create vault <app-version> <environment> --cluster <cluster>
```

After the deployment is up, the operator runs `vault operator init` against the external URL and then
unseals the server with the keys it prints. Keep the unseal keys and the root token outside Styrmin.

## Decisions

- **Agent injector off** (`injector.enabled: false`). The injector is a webhook that calls the Kubernetes
  API, which Styrmin's network policies block. Without it, pods do not get Vault Agent sidecars
  injected. Turn it on only when the Kubernetes API is reachable from the namespace.
- **authDelegator and serviceAccount.serviceDiscovery off.** The chart creates a ClusterRoleBinding and a
  Role for these. authDelegator serves the Kubernetes auth method, and serviceDiscovery labels HA pods.
  Both call the Kubernetes API, which Styrmin blocks. The standalone server uses neither. The check
  script warned about the service account's permissions until these were off.
- **Readiness uses the HTTP health endpoint** (`/v1/sys/health?standbyok=true&sealedcode=204&uninitcode=204`).
  The chart's default probe runs `vault status`, which exits 2 while Vault is sealed, so a new server
  would never become Ready. The chosen path returns 204 for a sealed or uninitialized server, so the
  pod becomes Ready and the URL answers while the operator unseals it.
- **Standalone mode, file storage at `/vault/data`.** The chart's `server.standalone` config is used
  unchanged. HA (Raft or Consul) is not deployed, so the component is `singleton: true`.
- **Ingress hosts**: `primary_fqdn` and `human_readable_fqdn`, both with path `/`. The ingress class is set
  by Styrmin.
- **Egress NetworkPolicy**: the chart's own `server.networkPolicy` carries an egress rule that allows
  DNS and traffic to pods in the same namespace. The chart's default ingress rule (port 8200 from all
  namespaces) is kept, so the ingress controller can reach the server.
- **Enterprise license not set.** The driver deploys the open source image. `server.enterpriseLicense`
  stays empty.
- **Backup kind `disk`** on the `data` volume. Not verified: whether a disk copy of a running file
  storage server restores a consistent Vault. Vault documents its own backup procedure (snapshots
  through the API); the driver does not use it.

## Checked

`check_driver.py --check-images` for app versions 2.0.0, 2.0.1 and 2.0.4: 0 FAIL, 0 WARN. The image
`hashicorp/vault` exists for linux/amd64 and linux/arm64 at each tag. The rendered manifests contain
the readiness path above, the Ingress host and path, and the NetworkPolicy.

## Not verified

- Not deployed on a cluster. The check script renders the chart with `helm template`; it does not
  start a pod.
- That Vault starts, reports Ready while sealed, and serves its API under Styrmin's network policies.
- That the disk backup restores a working Vault (see Decisions).
- The egress rule (DNS and the same namespace only) has not been tested against a running server.
- Upgrades from one application version to another, and the data migration they need.
