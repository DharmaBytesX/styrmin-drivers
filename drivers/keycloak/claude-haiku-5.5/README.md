# keycloak driver for Styrmin

Keycloak is an open source identity and access management server (https://github.com/keycloak/keycloak).
This driver deploys one Keycloak server, in plain HTTP mode behind the Styrmin ingress, with a
PostgreSQL database that the same chart installs. Keycloak uses the database for its realms, users
and clients. The driver takes a database dump backup of PostgreSQL.

| Component | Workload | Backup |
|---|---|---|
| `postgresql` | StatefulSet/styrmin-postgresql | postgres dump (database `bitnami_keycloak`) |
| `server` | StatefulSet/styrmin-keycloak | none |

- Chart: `oci://registry-1.docker.io/bitnamicharts/keycloak` version 25.2.0, which ships Keycloak 26.3.3. Source: https://github.com/bitnami/charts/tree/main/bitnami/keycloak
- Images: `docker.io/bitnamilegacy/keycloak:26.3.3-debian-12-r0` and `docker.io/bitnamilegacy/postgresql:17.6.0-debian-12-r4`. Both tags were confirmed to exist for linux/amd64 and linux/arm64 by `check_driver.py --check-images`.
- Supported application versions: `>=26.3.3,<26.3.4`. Only this version is checked (see decisions).
- External service: `keycloak`, HTTP on port 8080 of the `server` component. Users open `{{ instance.config.external_url_scheme }}://<primary_fqdn>`. The second host (`human_readable_fqdn`) is added to the Ingress as an extra host.

## How to load and deploy

```bash
styrminctl drivers load-version <git-url> <branch> --driver-directory keycloak-driver
styrminctl deployments create keycloak 26.3.3 <environment> --cluster <cluster>
```

## Credentials

These values are fixed in `values.j2.yml`. Every deployment of this driver gets the same passwords.

| Account | Where | Default |
|---|---|---|
| Keycloak administrator | `auth.adminUser`, `auth.adminPassword` | `admin` / `keycloak-admin-password` |
| Database user (Keycloak's own) | `postgresql.auth.username`, `postgresql.auth.password` | `bn_keycloak` / `keycloak-db-password` |
| Database superuser | `postgresql.auth.postgresPassword` | `keycloak-db-superuser-password` |

To change them, edit the values in `values.j2.yml` and load the driver again. The chart writes the
administrator password and the database passwords when the deployment is created. Changing the values
on an existing deployment does not change the stored passwords. That behavior is not verified here.
Change the defaults before any shared use.

## Parameters

None.

## Decisions

- **Chart source.** Artifact Hub has no chart published by the Keycloak project. The most starred
  verified chart is the Bitnami one, so the driver uses it. The index points the chart to its OCI
  location, so the driver uses that location.
- **Image tags.** Bitnami removed versioned tags from `docker.io/bitnami`. The driver uses the
  `docker.io/bitnamilegacy` copies of the same tags.
- **Image tag format.** The Keycloak tag is `{{ app.version }}-debian-12-r0`. The version range is
  one patch release because the `-debian-12-r0` suffix is only confirmed for 26.3.3. Widening the range
  needs each new tag checked with `check_driver.py --check-images --app-version <version>`.
- **Plain HTTP, not production mode.** `production: false` and `httpEnabled: true`. The Styrmin ingress
  terminates the connection, and Keycloak reads the forwarded headers (`proxyHeaders: xforwarded`) and
  builds its URLs from `ingress.hostname` (`hostnameStrict: true`). The chart's values say production mode needs TLS
  unless proxy headers are set. The driver sets proxy headers but does not configure TLS in Keycloak,
  and it sets `production: false`, so Keycloak does not run in production mode. This is not a hardened production setup.
- **Admin user name.** `admin` instead of the chart default `user`.
- **Network.** The egress policy allows DNS and traffic between the pods of the deployment. Keycloak
  needs no internet access, so the driver does not allow it. Keycloak can reach an external identity
  provider only if you add an egress rule for it.
- **Single instance.** Both components are `singleton: true`. The chart runs one replica of each.
  The database is standalone, with no read replicas.
- **Backup.** The `postgresql` component is backed up as a dump. The dump writes to a `backup` volume
  mounted at `/backup`, as the backup reference requires.
- **Icon.** The icon URL comes from the Bitnami stack page and is hosted on a third-party CDN. Replace it
  if you need a copy you control.

## Checked

`check_driver.py /work/keycloak-driver --check-images`: 0 FAIL, 0 WARN. The checks cover the spec
schema, the values template, `helm template` of chart 25.2.0 (17 resources), the components and their
StatefulSets, the external service, DNS reachability from every pod, the backup kind, and both image tags.

I also read the rendered manifests (`/tmp/check/manifests.yaml`):
- The Ingress has the primary host with path `/` and the human-readable host with path `/`.
- Keycloak uses `KC_DB_URL` `jdbc:postgresql://styrmin-postgresql:5432/bitnami_keycloak`, the same host as the database Service.
- The database volume uses the storage class from the check context, `standard-rwo`.

## Not verified

- The driver is not deployed on a cluster. The checks render the chart offline and do not run the pods.
- No login was tested: no administrator sign-in, no realm, no client.
- The backup was not run, so the dump and its restore are unverified.
- Changing the default passwords on an existing deployment is unverified (see Credentials).
- Other Keycloak versions and other chart versions are not verified.
