# keycloak driver for Styrmin

Deploys Keycloak, the identity and access management server, with a PostgreSQL database, from the Bitnami Keycloak Helm chart. Users reach the Keycloak console and realms on the deployment's two hosts (`primary_fqdn` and `human_readable_fqdn`).

| Component | Workload | Backup |
|---|---|---|
| `postgresql` | StatefulSet/styrmin-postgresql | postgres dump |
| `server` | StatefulSet/styrmin-keycloak | none (state is in the database) |

- Chart: `oci://registry-1.docker.io/bitnamicharts/keycloak` 25.2.0 (application 26.3.3).
- Images: `docker.io/bitnamilegacy/keycloak:26.3.3-debian-12-r0` and `docker.io/bitnamilegacy/postgresql:17.6.0-debian-12-r4`.
- Supported application versions: `>=26.3.3,<26.3.4`. Default: `26.3.3`.

## Load and deploy

```bash
styrminctl drivers load-version <git-url> <branch> --driver-directory keycloak-driver
styrminctl deployments create keycloak 26.3.3 <environment> --cluster <cluster>
```

## Credentials

Fixed in `values.j2.yml`. Change them there before you load the driver into production.

- Keycloak administrator: user `admin`, password `styrmin-keycloak-admin`.
- PostgreSQL user `bn_keycloak`, password `styrmin-keycloak-db`; `postgres` password `styrmin-keycloak-postgres`.

## Parameters

None.

## Decisions and reasons

- Chart choice: Keycloak publishes no Helm chart (it ships an operator). Artifact Hub lists no official chart, so I took the Bitnami chart: verified publisher, most stars, first in the list.
- Images: Bitnami removed versioned tags from `docker.io/bitnami`, so the driver uses `docker.io/bitnamilegacy` and sets `global.security.allowInsecureImages: true`, which the chart requires for a changed registry. The legacy images get no more updates; this is a security risk for long-lived deployments.
- Version range: the image tag ends in `-debian-12-r0`, which exists only for 26.3.3. The range is limited to that version, because other versions have different revision suffixes.
- Hostname: `hostnameStrict: false` and `proxyHeaders: xforwarded`, so that both hosts work behind the ingress controller. The chart still sets `KC_HOSTNAME` to the primary host.
- Plain HTTP inside the cluster (`httpEnabled: true`); the ingress controller terminates TLS.
- Single replica, `singleton: true` on both components: the chart's cache is not configured for clustering here.
- No internet egress rule in the NetworkPolicy. Add the commented rule if users need identity brokering with external providers or mail sending.
- Metrics and `keycloakConfigCli` are off; they are not needed.
- The chart's own NetworkPolicy stays on (default).

## Checked

`check_driver.py --check-images`: 0 FAIL, 0 WARN. Spec, template rendering, `helm template`, components, ingress hosts and paths, DNS egress, backup and image tags all passed. The supported range was narrowed after that run; the template is unchanged.

## Not verified

- Not deployed on a cluster; Keycloak startup within 270 s, login and the postgres dump backup are untested.
- The chart's own NetworkPolicy was not tested with Styrmin's ingress controller.
