# Test result: n8n driver by Claude Code, Claude Haiku 5.5

- Written by: Claude Code, Claude Haiku 5.5, with the `creating-styrmin-drivers` skill 0.1.10, in an isolated container.
- Chart: `oci://ghcr.io/n8n-io/n8n-helm-chart/n8n` 1.17.0; application version `2.44.5`.
- Tested on: Styrmin 0.1.54 on a local kind cluster (network policies enforced), on 2026-10-28.
- Load: from git, then deployed pinned to this driver version in its own environment.
- Result: **works**. Styrmin reported the deployment `running` after 90 s.
- HTTP checks through the ingress: `/healthz` 200, `/healthz/readiness` 200; admin login: not tested: the application creates its first user in the browser.
- Not tested: backups and restores, upgrades to another application version, a cloud cluster.
