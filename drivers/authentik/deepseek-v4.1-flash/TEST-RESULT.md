# Test result: authentik driver by opencode, GO plan, deepseek-v4.1-flash

- Written by: opencode, GO plan, deepseek-v4.1-flash, with the `creating-styrmin-drivers` skill 0.1.13, in an isolated container.
- Chart: `https://charts.goauthentik.io/authentik` 2026.8.5; application version `2026.8.5`.
- Tested on: Styrmin 0.1.54 on a local kind cluster (network policies enforced), on 2026-10-28.
- Load: from git, then deployed pinned to this driver version in its own environment.
- Result: **works**. Styrmin reported the deployment `running` after 140 s.
- HTTP checks through the ingress: `/-/health/ready/` 200; admin login: not ok through the ingress (see below).
- Admin login (akadmin with the bootstrap password) proven on a Helm install of the same rendered values in a scratch namespace, through a port-forward. Through the ingress, the test's login got a 302 on the flow requests; the cause was not found and is probably in the test.
- Not tested: backups and restores, upgrades to another application version, a cloud cluster.
