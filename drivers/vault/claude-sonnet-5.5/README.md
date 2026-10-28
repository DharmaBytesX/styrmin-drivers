# vault driver for Styrmin

Deploys HashiCorp Vault 2.0.x as a single server (standalone mode, file storage on a persistent volume) with the web UI enabled. Users open it at the deployment's URL.

| Component | Workload | Backup |
|---|---|---|
| `server` | StatefulSet/styrmin-vault | disk (volume `data`) |

- Chart: `https://helm.releases.hashicorp.com/vault` 0.34.1 (application 2.0.4), from the HashiCorp chart repository listed on Artifact Hub as official.
- Image: `hashicorp/vault:2.0.4`, checked in the registry for linux/amd64 and linux/arm64.

## Load and deploy

```bash
styrminctl drivers load-version <git-url> <branch> --driver-directory vault-driver
styrminctl deployments create vault 2.0.4 <environment> --cluster <cluster>
```

## Credentials

- The driver sets no password or token. Vault starts sealed and uninitialized.
- After the first deployment, an operator runs `vault operator init` against the URL. This prints the unseal keys and the initial root token. Store them outside Vault.
- Vault seals itself again after every pod restart (including a Styrmin stop and start). An operator must unseal it with `vault operator unseal`.

## Parameters

None.

## Decisions and reasons

- Standalone mode with file storage, not high availability with Raft. A single-primary store matches Styrmin's `singleton: true` and the `disk` backup of the `data` volume. Raft HA would need several pods, and the chart's HA mode uses the Kubernetes API for discovery.
- Agent injector and CSI provider are turned off. The injector calls the Kubernetes API and installs a cluster-wide MutatingWebhookConfiguration; Styrmin's network policies block that API access, and the check script flags such components. The injector component is therefore not in the spec.
- `server.authDelegator` and `server.serviceAccount.serviceDiscovery` are off. They create a ClusterRoleBinding and a role for Kubernetes auth and pod discovery, which a namespaced deployment does not need. Users cannot use Vault's Kubernetes auth method against the hosting cluster with this driver.
- The readiness probe is an HTTP call to `/v1/sys/health` that answers 204 when Vault is sealed or uninitialized. The chart's default probe runs `vault status`, which fails in those states, so the pod would never become Ready before an operator initializes it and Styrmin's 300 s limit would be exceeded.
- Web UI is on, so the external service shows a page.
- The chart's own NetworkPolicy (`server.networkPolicy`) carries the egress rules (DNS, and traffic inside the namespace), because the chart has no value for extra objects.
- Ingress has paths `/` on both hosts (`primary_fqdn` and `human_readable_fqdn`). The ingress class is not set; Styrmin sets it.
- `supported_versions` is `>=2.0,<2.1`: the chart's application version is 2.0.4. I did not check that other 2.0.x images work with this chart version; the image check covers only 2.0.4.
- No `actions.py`: initialization and unsealing are deliberately left to the operator, because they produce secrets that must not be stored by Styrmin.

## Checked

`check_driver.py --check-images`: 0 FAIL, 0 WARN. It validated the spec, rendered the template, ran `helm template` (8 resources), and checked components, services, ingress hosts, DNS egress, backup and the image tag.

## Not verified

- Not deployed on a cluster: pod readiness, ingress reachability and the disk backup were not run.
- Other 2.0.x application versions were not rendered or pulled.
