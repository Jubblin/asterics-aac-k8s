# couch-auth image notes

Based on reading `superlogin/start.js` in upstream `asterics/Asterics-AAC` (2026-08): it already loads its CouchDB and mailer settings from environment variables via `dotenv-flow`, so **no source patch is needed** to containerize it. Set these env vars on the Deployment (see `k8s/base/couchauth-deployment.yaml`):

| Env var | Purpose | Default in upstream code |
|---|---|---|
| `DB_SERVER_PUBLIC_URL` | Public URL of CouchDB (used in some redirect/link generation) | `http://127.0.0.1:5984` |
| `DB_SERVER_PROTOCOL` | `http://` or `https://` | `http://` |
| `DB_SERVER_HOST` | `host:port` of CouchDB | `127.0.0.1:5984` |
| `DB_SERVER_USER` | CouchDB admin user | `admin` |
| `DB_SERVER_PASSWORD` | CouchDB admin password | `admin` |
| `CAUTH_USER_DB` | Database couch-auth uses for user profiles | `auth-users` |
| `CAUTH_COUCH_AUTH_DB` | CouchDB `_users`-style auth database | `_users` |
| `MAILER_FROM_EMAIL`, `MAILER_HOST`, `MAILER_PORT`, `MAILER_SECURE`, `MAILER_USER`, `MAILER_PASS` | SMTP settings for account emails | unset |

## Important caveat: email is stubbed by default

The upstream `config` object hardcodes:
```js
testMode: { noEmail: true, ... },
local: { sendConfirmEmail: false, requireEmailConfirm: false, ... }
```
These are **not** currently env-driven, meaning even if you set `MAILER_*`, no confirmation/password-reset emails are actually sent — registration works without email confirmation. This is fine for a first deployment (accounts just work), but if you need real email flows (password reset in particular), you'll need to fork `superlogin/start.js` and either wire those three fields to env vars yourself or hardcode them off `testMode`. Track this as a follow-up; not blocking initial deployment.

## TLS

`useSSL` is hardcoded `false` in upstream `start.js` — the process always listens plain HTTP on port 3000. That's what we want here: TLS is terminated at the Kubernetes Ingress, not in the pod.

## Session/security config

`@klues/couch-auth`'s own internal session/token config isn't touched by this deployment — it uses the library's defaults. Review the [couch-auth docs](https://github.com/perfood/couch-auth) if you need to override session TTL, token secret, etc., before relying on this in production.
