# jenkins driver for Styrmin

Deploys Jenkins (the automation server) as a single controller StatefulSet using the official
[`jenkinsci/jenkins` Helm chart](https://github.com/jenkinsci/helm-charts) (5.10.0, application
2.580.1). The controller serves the Jenkins web UI over HTTP on port 8080 and is exposed through an
Ingress; its `jenkins_home` (jobs, configuration, plugins) is kept on a PersistentVolume backed by
the environment's standard storage class.

| Component | Workload | Container | Backup |
|---|---|---|---|
| `jenkins-controller` | StatefulSet/styrmin-jenkins | `jenkins` | disk (volume `jenkins-home`) |

- Chart: `https://charts.jenkins.io/jenkins` 5.10.0 (application 2.580.1).
- Images: `docker.io/jenkins/jenkins:2.580.1-jdk25` (controller and its init container).

## Load and deploy

```bash
styrminctl drivers load-version <git-url> <branch> --driver-directory jenkins-driver
styrminctl deployments create jenkins 2.580.1 <environment> --cluster <cluster>
```

## Credentials

- Admin user: `admin`
- Admin password: `jenkins-admin-password` (fixed, not generated, so a restored or cloned
  deployment still matches its data)

Change them by editing `controller.admin.username` / `controller.admin.password` in `values.j2.yml`
before loading the driver. The chart stores them in the `styrmin-jenkins` Secret and seeds Jenkins
through its JCasC security realm on first start.

## Parameters

None.

## Decisions and why

- **`fullnameOverride: styrmin-jenkins`** — the Helm release name is `styrmin-<36-char uuid>`
  (44 characters). The chart names the RBAC `Role`/`RoleBinding` as `<fullname>-schedule-agents`;
  with the release-name-based full name that is 68 characters, which Kubernetes rejects. The short
  override keeps every resource name within limits.
- **`agent.enabled: false` + `controller.numExecutors: 2`** — Styrmin puts `default-deny-egress`
  in every deployment namespace and the policy shipped by this driver does not reach the Kubernetes
  API, so the Jenkins Kubernetes cloud cannot spawn agent pods. Disabling the agent cloud removes
  dead configuration (and a pod label the chart builds from the release name), and running two
  executors on the controller keeps Jenkins usable for builds.
- **`controller.sidecars.configAutoReload.enabled: false`** — the `k8s-sidecar` watches ConfigMaps
  through the Kubernetes API, which Styrmin blocks. JCasC configuration is still applied on startup
  by the chart's `apply_config.sh` init script.
- **`controller.jenkinsUrlProtocol` = the external scheme** — Styrmin terminates TLS at its ingress
  controller, so Jenkins must announce an `https` URL even though the chart's Ingress carries no TLS
  block; this makes redirects and absolute links use the right scheme.
- **Ingress hosts** — the chart's Ingress takes a single primary host (`controller.ingress.hostName`)
  plus a second host via `controller.ingress.resourceRootUrl`, which is only used by the Ingress
  template (not by Jenkins' own configuration). Both hosts forward to the same controller Service on
  port 8080. `primary_fqdn` is the primary host and `human_readable_fqdn` is the second alias.
- **Egress NetworkPolicy** — allows DNS, the deployment's own pods, and the internet (excluding the
  private ranges). Jenkins needs the internet on first start to download plugins and later for SCM
  checkouts in builds.
- **`controller.image.tag: "<app.version>-jdk25"`** — the registry tags the controller image as
  `<version>-jdk25` (the chart's `tagLabel`). Only `2.580.1` has a matching image, so
  `supported_versions` is narrowed to that single patch release.

## Checked

`check_driver.py jenkins-driver --check-images` reports **0 FAIL, 0 WARN**. The image
`docker.io/jenkins/jenkins:2.580.1-jdk25` exists for `linux/amd64` and `linux/arm64`. Rendering the
chart with the driver's values produces the StatefulSet, Services, Ingress (both FQDNs), PVC and the
egress NetworkPolicy, and the component labels match the StatefulSet and its pods.

## Not verified

- Not deployed on a live cluster; only the offline checks above were run.
- The Kubernetes agent cloud is disabled because Styrmin's egress policy blocks the Kubernetes API;
  builds run on the controller executors. This behaviour has not been exercised on a cluster.
