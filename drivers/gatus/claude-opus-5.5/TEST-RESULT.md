# Test result: gatus driver by Claude Code, Claude Opus 5.5

- Written by: Claude Code, Claude Opus 5.5, with the `creating-styrmin-drivers` skill 0.1.12, in the main Claude Code session that runs this bake-off (not isolated: it had seen the other runs).
- Chart: `oci://ghcr.io/bjw-s-labs/helm/app-template` 5.2.1; application version `5.34.0`.
- Tested on: Styrmin 0.1.54 on a local kind cluster (network policies enforced), on 2026-10-28.
- Load: from git, then deployed pinned to this driver version in its own environment.
- Result: **works**. Styrmin reported the deployment `running` after 80 s.
- HTTP checks through the ingress: `/health` 200, `/api/v1/endpoints/statuses` 200; admin login: not tested: the application creates its first user in the browser.
- Not tested: backups and restores, upgrades to another application version, a cloud cluster.
