# AsTeRICS AAC on Kubernetes — Deployment Plan

Source: [asterics/Asterics-AAC](https://github.com/asterics/Asterics-AAC) (AGPL-3.0). No official Docker image or Helm chart exists upstream — everything in this repo builds that from scratch.

## 1. What actually needs deploying

AsTeRICS AAC is a Vue.js PWA (`webpack` build → static `app/build` folder). Almost everything runs client-side (grids, TTS, IndexedDB storage). Only two features need server-side infrastructure:

| Component | Needed for | Upstream tech |
|---|---|---|
| **Frontend static app** | The app itself — always required | Vue 2, webpack, served as static files |
| **couch-auth (superlogin)** | "Online users" — account login, cross-device sync | `superlogin/start.js`, Node/Express, `@klues/couch-auth` |
| **CouchDB** | Backing store for online users (one DB per user), synced via PouchDB replication | Apache CouchDB |
| **Matrix homeserver** *(optional)* | Built-in messenger/chat feature | Any Matrix server (Synapse/Dendrite) — can point at an existing homeserver instead of self-hosting |

Everything else is a third-party API the app calls directly from the browser — **not self-hosted**: ARASAAC, opensymbols.org, radio-browser.info, podcastindex.org, open-meteo.com, YouTube, ResponsiveVoice. These just need egress allowed from the client's network, nothing to deploy.

## 2. Containerization notes

- `superlogin/start.js` hardcodes CouchDB connection details around line 38 upstream. `docker/couchauth/Dockerfile` in this repo expects that file patched (or overridden at runtime) to read `COUCHDB_URL`, `COUCHDB_USER`, `COUCHDB_PASSWORD`, session secret, and SMTP settings from environment variables via `dotenv-flow` (already a dependency) — see `docker/couchauth/README.md` for the exact patch.
- The frontend's PWA service worker (`serviceWorker.js`) must be served with `Cache-Control: no-cache` so clients pick up updates — handled in `docker/frontend/nginx.conf`.
- CouchDB itself uses the unmodified official `couchdb:3` image.

## 3. Topology

```
Ingress (TLS via cert-manager)
 ├─ /            → Service: aac-frontend   → Deployment (nginx, static build)
 ├─ /auth        → Service: aac-couchauth  → Deployment (Node/Express)
 └─ (matrix.*)   → external or separately-deployed homeserver [optional]
                                │
                        Service: couchdb    → StatefulSet (PVC)
```

## 4. Cross-cutting concerns

- **NetworkPolicy**: default-deny in the namespace, then explicitly allow frontend→couch-auth and couch-auth→couchdb (`k8s/base/networkpolicy.yaml`). Calls to third-party APIs (ARASAAC, YouTube, etc.) happen from the end user's browser, not from pods, so they don't need cluster egress rules.
- **Backups**: CouchDB PVC needs scheduled backups (e.g. a `couchbackup` CronJob dumping to object storage) since it holds every online user's grids. Not included in this scaffold yet.
- **PodDisruptionBudget**: included for CouchDB and couch-auth (`k8s/base/pdb.yaml`) so cluster maintenance doesn't take auth/sync down entirely.
- **CDN**: since the frontend is 100% static, consider a CDN in front of the Ingress instead of relying purely on pod replicas for scale.

## 5. Verification checklist

- [ ] `docker build` succeeds for both images; `npm run test` (Jest) passes in the frontend build stage.
- [ ] Frontend loads over HTTPS with a valid cert.
- [ ] Service worker registers and the app works offline after first load.
- [ ] Account registration email arrives (SMTP config correct) and login/sync works end-to-end.
- [ ] CouchDB PVC survives pod restart (`kubectl delete pod couchdb-0` and confirm data persists).
- [ ] NetworkPolicy doesn't block legitimate frontend↔couch-auth↔couchdb traffic (test before enabling default-deny in prod).
- [ ] Backup job produces a restorable dump.
