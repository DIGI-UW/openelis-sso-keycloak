# OpenELIS Global with Keycloak single sign-on

OpenELIS Global 3.3.0 (the [openelis-docker](https://github.com/DIGI-UW/openelis-docker) 3.3.0 stack)
with [Keycloak](https://www.keycloak.org) 26 as a SAML 2.0 identity provider. Users click
**Single Sign On** on the OpenELIS login page, sign in at Keycloak and come back logged in to OpenELIS
with the roles Keycloak gives them.

Guide: [Single Sign-On (SSO) with Keycloak](https://uwdigi.atlassian.net/wiki/spaces/oeg/pages/1776844808)

## Start

```bash
docker compose up -d
```

The first start takes a few minutes. Then open:

| Service | URL | User : password |
|---|---|---|
| OpenELIS | <https://localhost> | `admin` : `adminADMIN!` (local login) |
| OpenELIS with SSO | <https://localhost>, **Single Sign On** | `jdoe` : `password` |
| Keycloak admin console | <https://localhost/auth/admin> | `admin` : `KEYCLOAK_ADMIN_PASSWORD` from `.env` |

Change `jdoe`'s password and the Keycloak admin password before anyone else can reach the server.

## How it is wired

- Keycloak runs as `keycloak.openelis.org` and has no published port. Browsers reach it through the
  OpenELIS proxy at `https://<host>/auth` (`location /auth/` in `configs/nginx/nginx.conf`).
  Keycloak must be on the same HTTPS site as OpenELIS: when Keycloak posts the SAML response back, the
  browser sends the OpenELIS session cookie only on a same-site request. On another host or port the login
  fails with `invalid_in_response_to`.
- OpenELIS reads the Keycloak SAML metadata once, at startup, over the internal network
  (`org.itech.login.saml.metadatalocation` in `configs/properties/common.properties`). The webapp
  therefore waits until Keycloak is healthy.
- The realm `OpenELIS` in `configs/keycloak/realm-export.json` is imported on the first start only. It has
  the SAML client `OpenELIS-Global_saml`, the client roles (`oeg-Global Administrator`,
  `oeg-Reception-AllLabUnits`, ...) and the user `jdoe`. Later changes are made in the Keycloak console
  and kept in the `keycloak-data` volume.
- OpenELIS creates the user on the first SSO login and gives it the roles of the SAML attribute `Role`
  (values prefixed `oeg-`), on every login.

## Deploy on your own host name

1. In `.env`, set `KEYCLOAK_URL=https://<your host>/auth` (add `:<port>` when `OE_HTTPS_PORT` is not 443).
2. In the Keycloak console, realm **OpenELIS** > Clients > `OpenELIS-Global_saml`: set **Valid redirect
   URIs** and the **Master SAML Processing URL** to your host (the import allows only `https://localhost/*`).
3. `docker compose up -d`, then `docker compose up -d --force-recreate oe.openelis.org` so OpenELIS reads
   the metadata again.

## Verified with this setup

On OpenELIS 3.3.0 and Keycloak 26.4.5: SSO login as `jdoe`, user creation in OpenELIS with the
Keycloak roles, a role change in Keycloak applied at the next login, and SSO logout.

Everything else (backups, upgrades, certificates, light stack) works as in openelis-docker: see its
[README](https://github.com/DIGI-UW/openelis-docker#readme).
