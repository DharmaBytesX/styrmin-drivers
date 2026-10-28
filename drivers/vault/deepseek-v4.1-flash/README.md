# vault driver for Styrmin

Deploys [HashiCorp Vault](https://www.vaultproject.io) as a single-node secrets manager on
Kubernetes with the official HashiCorp Helm chart. The server runs in `standalone` mode with the
file storage backend on a PersistentVolumeClaim, and the Vault UI/API is published on the
deployment URL through the chart's Ingress. The Agent Injector and the CSI provider are disabled,
because both need the Kubernetes API that Styrmin's default-deny egress policy blocks.

| Component | Workload | Backup |
|---|---|---|
| `server` | StatefulSet/styrmin-vault (container `vault`) | disk snapshot of the `data` volume |

- Chart: `https://helm.releases.hashicorp.com/vault` 0.34.1 (application 2.0.4).
- Image: `hashicorp/vault:2.0.4` (tag rendered from the deployment's application version).
- Supported application versions: `>=2.0,<=2.0.4` (default `2.0.4`).
- Parameters: none.

## Load and deploy

```bash
styrminctl drivers load-version <git-url> <branch> --driver-directory <path/to>/vault-driver
styrminctl deployments create vault 2.0.4 <environment> --cluster <cluster>
```

## First start: initialize and unseal

Vault is **sealed** after every start until it is initialized (once) and unsealed. There is no
sane automatic unseal without an external KMS (Vault's auto-unseal needs AWS/GCP/Azure/transit),
so this driver leaves initialization to the operator. The chart's readiness probe is disabled on
purpose: it runs `vault status`, which fails while sealed, and FluxCD would roll the install back
at 300 s. With the probe off, the pod becomes Ready and the UI (which serves the unseal page) is
reachable at the deployment URL.

Initialize once, from the pod (the chart sets `VAULT_ADDR=http://127.0.0.1:8200`):

```bash
kubectl -n <namespace> exec -it styrmin-vault-0 -- vault operator init
```

Save the output: it contains the **root token** and the **unseal keys**. There is no default
password or token; the root token is whatever `vault operator init` prints, and it is not stored
in the chart values. Keep the init output outside the cluster, then unseal with any three of the
five keys (the default share/threshold):

```bash
kubectl -n <namespace> exec -it styrmin-vault-0 -- vault operator unseal <key-1>
kubectl -n <namespace> exec -it styrmin-vault-0 -- vault operator unseal <key-2>
kubectl -n <namespace> exec -it styrmin-vault-0 -- vault operator unseal <key-3>
```

The same steps are available in the UI. After every pod restart, unseal again. Log in with the
root token and create real auth methods and policies; the root token is the break-glass
credential.

To rotate credentials, use Vault itself (`vault operator generate-root`, `vault token revoke`,
`vault operator rekey`). Nothing in this driver generates or stores Vault credentials.

## Backup and restore

The `server` component uses a `disk` backup of the `data` volume (`/vault/data`). The data is
encrypted with Vault's master key, which is protected by the unseal keys. A restore is only
usable together with the matching unseal keys (or the same auto-unseal configuration), so store
the initialization output separately from the cluster and its backups.

## Decisions

- **Standalone + file storage, one replica.** Vault's dev mode would auto-unseal but keeps all
  data in memory, which is unacceptable for a secrets manager. Standalone with a PVC persists
  secrets and allows a volume backup.
- **Readiness and liveness probes off.** A sealed Vault fails `vault status`; keeping the probe
  would make the pod never Ready and cause a FluxCD rollback before anyone could initialize it.
- **Agent Injector and CSI provider off.** They are extra workloads that talk to the Kubernetes
  API, and Styrmin denies that egress in the deployment namespace.
- **`server.authDelegator.enabled: false`.** It would bind the server's service account to
  `system:auth-delegator` for Vault's Kubernetes auth method, but that method needs Kubernetes
  API egress, which Styrmin blocks. As a result the Kubernetes auth method is not available out
  of the box; enable the delegator and add an egress rule for the API server if you need it.
- **`ui.enabled` stays false.** The server listener already serves the UI (`ui = true` in the
  standalone config) on port 8200; the chart value only creates a second Service, which is not
  needed.
- **Ingress from the chart.** Both `services['vault'].primary_fqdn` and
  `human_readable_fqdn` are configured as hosts; empty `paths` makes the chart use `/`. The
  ingress class is left to Styrmin.
- **NetworkPolicy from the chart.** `server.networkPolicy` is enabled with egress for DNS
  (kube-dns/coredns) and for same-namespace pods, plus the chart's default ingress rule for
  Vault's ports.
- **Versions capped at 2.0.4.** The chart's current appVersion is 2.0.4 and registry tags
  2.0.5/2.0.6 do not exist yet, so the range stops there rather than accepting a version that
  cannot be pulled.
- **No parameters and no actions.** The single-node model has nothing safe to vary, and
  init/unseal involves secret material that an action must not log; the manual procedure above
  is deliberate.

## Checked

`scripts/check_driver.py /work/vault-driver --check-images` reports **0 FAIL, 0 WARN** for the
default version 2.0.4, including registry checks for `hashicorp/vault:2.0.4` on linux/amd64 and
linux/arm64. Versions 2.0.0, 2.0.1, 2.0.2 and 2.0.3 were also rendered and their images verified
(0 FAIL, 0 WARN). `helm template` renders a StatefulSet, two Services, a ConfigMap, a
ServiceAccount, a NetworkPolicy and the Ingress with both hosts and `/` paths.

## Not verified

- Not deployed on a real cluster: this was checked offline with `helm template` and the registry.
- Initialization, unsealing, UI login, Ingress routing and the disk backup/restore were not
  exercised against a live Vault.
- The chart NetworkPolicy assumes CoreDNS is labelled `k8s-app` with `kube-dns` or `coredns`;
  a cluster that labels it differently needs an extra egress rule.
