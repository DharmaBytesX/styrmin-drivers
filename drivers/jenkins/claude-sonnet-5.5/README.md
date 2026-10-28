# jenkins driver for Styrmin

Deploys the Jenkins controller (the automation server) from the official chart. The controller runs as one StatefulSet with a persistent volume for the Jenkins home directory. Users open it on the deployment's primary host and on a second, human-readable host. The default chart plugins (git, workflow-aggregator, configuration-as-code, kubernetes) are installed at start.

| Component | Workload | Backup |
|---|---|---|
| `jenkins-controller` | StatefulSet/styrmin-jenkins | disk |

- Chart: `https://charts.jenkins.io/jenkins` 5.10.0 (application 2.580.1).
- Credentials: user `admin`, password `styrmin-jenkins-admin` (`controller.admin.password` in `values.j2.yml`). The password is fixed because Styrmin needs fixed credentials. Change it in `values.j2.yml` before use in production; the value is applied by the Jenkins Configuration as Code plugin when a new home volume is first created.
- Parameters: none.
- Decisions:
  - Chart: the official `jenkinsci` chart, not the Bitnami one, because it is maintained by the Jenkins project and ships the current LTS.
  - Image tag: `{{ app.version }}-jdk25`, because the chart's default tag adds the `jdk25` label and the tag is checked in the registry. Supported versions are limited to `>=2.580,<2.581` because only that version was checked against this chart and tag; other versions are not tested.
  - The `config-reload` sidecar and its init container are turned off (`controller.sidecars.configAutoReload.enabled: false`) because they call the Kubernetes API, which Styrmin's network policies block. Effect: changes to the Configuration as Code ConfigMap apply only after the pod restarts.
  - The default Kubernetes agent configuration is off (`agent.enabled: false`) because the kubernetes plugin starts agent pods through the Kubernetes API, which is blocked. Builds run on the controller only, and `numExecutors` is the chart default of 0, so users must add executors or external agents (agents connect on port 50000, which is not exposed outside the namespace) before jobs run. This was not changed because it is a choice of the Jenkins owner.
  - The egress NetworkPolicy allows DNS, traffic inside the namespace and the internet (except private ranges), because the init container downloads plugins from the Jenkins update site at each start. Without internet access the pod does not start.
  - The second host uses the chart's `secondaryingress` with path `/`, because the primary ingress takes one host.
  - Backup: kind `disk` on the `jenkins-home` volume, singleton component, because Jenkins keeps all state in its home directory.
  - Check WARN about a service account with Kubernetes permissions (RoleBinding `schedule-agents`): the Role is only used by the kubernetes plugin to start agents, which is off, and no container calls the API at start. Not fixed because the chart always creates it.
- Checked: `check_driver.py --check-images` reports 0 FAIL and 1 WARN (the service account WARN explained above). The image tag exists for linux/amd64 and linux/arm64.
- Not verified: the driver was not deployed on a cluster. Not known: whether plugin download plus start fits within the 270 s readiness limit (the chart's startup probe allows 120 s after the init container); if pods are not Ready in time, raise `controller.probes.startupProbe.failureThreshold`.
