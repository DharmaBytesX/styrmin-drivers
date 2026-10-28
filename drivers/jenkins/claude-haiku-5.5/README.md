# jenkins driver for Styrmin

This driver deploys Jenkins, the automation server (https://github.com/jenkinsci/jenkins), on a Styrmin
Kubernetes deployment. It uses the official Jenkins Helm chart (https://charts.jenkins.io/jenkins). A user
opens the Jenkins web interface at the deployment's URL, signs in with the admin account below, and creates
jobs and pipelines on the controller.

| Component | Workload | Backup |
|---|---|---|
| `jenkins-controller` | StatefulSet/styrmin-jenkins | disk (volume `jenkins-home`) |

- Chart: `https://charts.jenkins.io/jenkins` 5.10.0 (application 2.580.1). Source: https://github.com/jenkinsci/jenkins
- Images: `docker.io/jenkins/jenkins:2.580.1-jdk25`. The check script confirmed the tag exists for linux/amd64 and linux/arm64.
- Service: `jenkins` on port 8080, served from the deployment's primary host through the chart's Ingress.
- Storage: one persistent volume for `/var/jenkins_home` (the Jenkins home directory), in the standard storage class.
- Network: an egress NetworkPolicy allows DNS, traffic between the pods of the deployment, and traffic to the internet outside private address ranges. Styrmin denies all other egress.

## Loading and deploying

```bash
styrminctl drivers load-version <git-url> <branch> --driver-directory jenkins-driver
styrminctl deployments create jenkins 2.580.1 <environment> --cluster <cluster>
```

## Credentials

- Admin user: `admin`. Password: `styrmin-jenkins-admin`.
- The password is fixed in `values.j2.yml` (`controller.admin.password`). Anyone who can read this driver can read it. To change it, edit that value and load the driver again. Changing the value in a running deployment is not tested.

## Parameters

None.

## Decisions

- **The config-reload sidecars are off.** The chart runs two containers (`config-reload-init` and `config-reload`, image `kiwigrid/k8s-sidecar`) that watch ConfigMaps through the Kubernetes API. Styrmin's network policies block that API traffic, so both containers would never become ready. Rule 16 in the creating-styrmin-drivers skill covers this case, and the chart's `controller.sidecars.configAutoReload.enabled` turns them off.
- **Jenkins security is set by a Groovy init script, not by Configuration as Code.** The chart delivers its Configuration as Code (JCasC) files through those same sidecars. With the sidecars off, the chart's JCasC files never reach Jenkins, so the driver sets the security realm, the admin account and the authorization strategy with `controller.initScripts.security`. Jenkins runs that script at every start. The script:
  - creates the admin account from the chart's admin secret,
  - allows signed-in users full control and blocks anonymous access,
  - sets the Jenkins URL to the deployment's URL,
  - marks the setup wizard as complete.
  - Not known: whether the script runs without error on Jenkins 2.580.1. The Groovy script was not run here, because no Java or Groovy runtime is installed in this environment. Check the controller's log after the first start.
- **The chart's RBAC is off (`rbac.create: false`).** The chart grants the controller permission to manage pods for the Jenkins Kubernetes plugin. That plugin calls the Kubernetes API, which Styrmin blocks. The chart set up the Kubernetes cloud through its JCasC files, which this driver does not deliver, so the controller does not need those permissions. The controller's ServiceAccount is still created.
- **Internet egress is allowed.** The chart's init container downloads its default plugins (kubernetes, workflow-aggregator, git and others) from `updates.jenkins.io` each time the controller starts. The egress policy allows `0.0.0.0/0` except the private ranges `10/8`, `172.16/12`, `192.168/16` and `100.64/10`. Jobs that fetch code from the internet also need this rule. Without it, the plugins would not install.
- **Build agents run on the controller only.** The chart's default agents start pods through the Kubernetes API, which Styrmin blocks, so they are not configured. The chart's agent settings stay at their defaults but have no effect. Builds run on the controller's built-in node. Agents that connect from outside the cluster would need the JNLP port (50000), which this driver does not expose.
- **Ingress host only.** The chart's Ingress takes one host, the deployment's primary host. The chart has no value for a second host, so the `human_readable_fqdn` is not used.
- **Fixed admin password.** Styrmin requires fixed credentials, not generated ones (rule 4 in the skill).
- **Supported versions.** `supported_versions` is `>=2.580,<2.581`, because chart 5.10.0 ships Jenkins 2.580.1. Other Jenkins versions need their own chart version and driver version.

## Checked

`check_driver.py --check-images` reports 0 FAIL and 0 WARN. It validated the driver against the schema, rendered `values.j2.yml`, ran `helm template` on chart 5.10.0 with the rendered values, checked the component, the external service, the Ingress host, the DNS egress, the backup and the image tag. The rendered manifests were in `/tmp/check/manifests.yaml`.

## Not verified

- Not deployed on a Styrmin cluster. The checks above are offline.
- The Groovy security script was not run against Jenkins (see Decisions).
- Plugins: not installed in a running deployment. The check only confirmed that the internet rule renders. Whether `updates.jenkins.io` answers through the Styrmin network and whether each default plugin installs is not known.
- Jenkins start-up time: the chart's probes allow the default start-up time. Not measured.
- The URL port: the Jenkins URL is built from the primary host and the scheme only. A deployment that serves a non-default port would need `external_url_port` added to the URL in `values.j2.yml` and in the Groovy script.
