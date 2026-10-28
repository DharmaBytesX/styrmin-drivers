# Styrmin drivers

Application drivers for [Styrmin](https://docs.styrmin.app/).

- `driver.styrmin.yml` and `values.j2.yml` at the repository root: the generic-app driver (any
  single-container HTTP application), unchanged.
- `drivers/<application>/<model>/`: drivers written by AI models with the
  `creating-styrmin-drivers` skill, each in an isolated container, then deployed and tested.

## Tested drivers

Each driver below was loaded from git into Styrmin 0.1.54 on a local kind cluster with network
policies enforced, deployed in its own environment, and checked through its URL: a health endpoint
that reaches the database, and a login with the admin credentials the driver sets. Two
applications take no admin from configuration, so no login was tested: n8n creates its first user
in the browser, and a new Vault waits for its operator to initialize and unseal it. Every driver
passed. Its folder has a `TEST-RESULT.md` with the details.

| Application | Driver directory | Written by | Skill | Chart | App version | Running after |
|---|---|---|---|---|---|---|
| Gitea | `drivers/gitea/qwen3.8-27b-local` | opencode, local Qwen3.8 27B (MTPLX) | 0.1.7 | `https://dl.gitea.com/charts/gitea` 12.7.0 | 1.27.0 | 90 s |
| Grafana | `drivers/grafana/claude-haiku-5.5` | Claude Code, Claude Haiku 5.5 | 0.1.8 | `https://grafana-community.github.io/helm-charts/grafana` 14.2.0 | 13.3.0 | 90 s |
| Grafana | `drivers/grafana/claude-sonnet-5.5` | Claude Code, Claude Sonnet 5.5 | 0.1.8 | `https://grafana-community.github.io/helm-charts/grafana` 14.2.0 | 13.3.0 | 91 s |
| Grafana | `drivers/grafana/deepseek-v4-pro` | opencode, GO plan, deepseek-v4-pro | 0.1.8 | `https://grafana-community.github.io/helm-charts/grafana` 14.2.0 | 13.3.0 | 90 s |
| Grafana | `drivers/grafana/gpt-5.6-luna` | opencode, GO plan, gpt-5.6-luna | 0.1.8 | `https://grafana-community.github.io/helm-charts/grafana` 14.2.0 | 13.3.0 | 80 s |
| Grafana | `drivers/grafana/minimax-m3` | opencode, GO plan, minimax-m3 | 0.1.8 | `https://grafana-community.github.io/helm-charts/grafana` 14.2.0 | 13.3.0 | 80 s |
| Grafana | `drivers/grafana/qwen3.8-27b-local` | opencode, local Qwen3.8 27B (MTPLX) | 0.1.10 | `https://grafana-community.github.io/helm-charts/grafana` 14.2.0 | 13.3.0 | 81 s |
| Grafana | `drivers/grafana/qwen3.8-max` | opencode, GO plan, qwen3.8-max | 0.1.8 | `https://grafana.github.io/helm-charts/grafana` 10.5.15 | 12.3.1 | 100 s |
| Jenkins | `drivers/jenkins/claude-haiku-5.5` | Claude Code, Claude Haiku 5.5 | 0.1.9 | `https://charts.jenkins.io/jenkins` 5.10.0 | 2.580.1 | 100 s |
| Jenkins | `drivers/jenkins/claude-sonnet-5.5` | Claude Code, Claude Sonnet 5.5 | 0.1.9 | `https://charts.jenkins.io/jenkins` 5.10.0 | 2.580.1 | 80 s |
| Jenkins | `drivers/jenkins/deepseek-v4-pro` | opencode, GO plan, deepseek-v4-pro | 0.1.8 | `https://charts.jenkins.io/jenkins` 5.10.0 | 2.580.1 | 80 s |
| Keycloak | `drivers/keycloak/claude-haiku-5.5` | Claude Code, Claude Haiku 5.5 | 0.1.10 | `oci://registry-1.docker.io/bitnamicharts/keycloak` 25.2.0 | 26.3.3 | 140 s |
| Keycloak | `drivers/keycloak/claude-sonnet-5.5` | Claude Code, Claude Sonnet 5.5 | 0.1.10 | `oci://registry-1.docker.io/bitnamicharts/keycloak` 25.2.0 | 26.3.3 | 151 s |
| n8n | `drivers/n8n/claude-haiku-5.5` | Claude Code, Claude Haiku 5.5 | 0.1.10 | `oci://ghcr.io/n8n-io/n8n-helm-chart/n8n` 1.17.0 | 2.44.5 | 90 s |
| n8n | `drivers/n8n/claude-sonnet-5.5` | Claude Code, Claude Sonnet 5.5 | 0.1.10 | `oci://ghcr.io/n8n-io/n8n-helm-chart/n8n` 1.17.0 | 2.44.5 | 90 s |
| n8n | `drivers/n8n/gpt-5.6-luna` | opencode, GO plan, gpt-5.6-luna | 0.1.10 | `oci://ghcr.io/n8n-io/n8n-helm-chart/n8n` 1.17.0 | 2.44.5 | 90 s |
| NetBox | `drivers/netbox/gpt-5.6-luna` | opencode, GO plan, gpt-5.6-luna | 0.1.0 | `https://charts.netbox.oss.netboxlabs.com/netbox` 8.3.97 | 4.7.3 | 490 s |
| NetBox | `drivers/netbox/qwen3.8-27b-local` | opencode, local Qwen3.8 27B (MTPLX) | 0.1.5 | `https://charts.netbox.oss.netboxlabs.com/netbox` 8.3.97 | 4.7.3 | 524 s |
| Nextcloud | `drivers/nextcloud/claude-haiku-5.5` | Claude Code, Claude Haiku 5.5 | 0.1.9 | `https://nextcloud.github.io/helm/nextcloud` 9.4.1 | 35.0.2 | 140 s |
| Nextcloud | `drivers/nextcloud/claude-sonnet-5.5` | Claude Code, Claude Sonnet 5.5 | 0.1.10 | `https://nextcloud.github.io/helm/nextcloud` 9.4.1 | 35.0.2 | 91 s |
| Nextcloud | `drivers/nextcloud/deepseek-v4.1-flash` | opencode, GO plan, deepseek-v4.1-flash | 0.1.10 | `https://nextcloud.github.io/helm/nextcloud` 9.4.1 | 35.0.2 | 80 s |
| Nextcloud | `drivers/nextcloud/gpt-5.6-luna` | opencode, GO plan, gpt-5.6-luna | 0.1.10 | `https://nextcloud.github.io/helm/nextcloud` 9.4.1 | 35.0.2 | 151 s |
| Vault | `drivers/vault/claude-sonnet-5.5` | Claude Code, Claude Sonnet 5.5 | 0.1.11 | `https://helm.releases.hashicorp.com/vault` 0.34.1 | 2.0.4 | 100 s |
| Vault | `drivers/vault/deepseek-v4.1-flash` | opencode, GO plan, deepseek-v4.1-flash | 0.1.11 | `https://helm.releases.hashicorp.com/vault` 0.34.1 | 2.0.4 | 90 s |

Several drivers exist for the same application: they are alternatives written by different models.
They share the driver name (for example `grafana`), so Styrmin treats them as versions of one
driver: pin the one you want with `--driver-version-id`.

Not tested for any driver: backups and restores, upgrades to another application version, and
clusters other than kind.

## Load and deploy

```bash
styrminctl drivers load-version https://github.com/DharmaBytesX/styrmin-drivers.git main \
  --driver-directory drivers/grafana/claude-sonnet-5.5
styrminctl deployments create grafana <app-version> <environment> --cluster <cluster> \
  --driver-version-id <id printed by load-version>
```

## Before a real deployment

The drivers set fixed credentials in `values.j2.yml` (admin passwords, database passwords, secret
keys), so that a restored or cloned deployment still matches its data. These values are public in
this repository: replace them with your own before you deploy anything that matters.
