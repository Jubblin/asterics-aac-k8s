# asterics-aac-k8s

Kubernetes deployment for [AsTeRICS AAC](https://github.com/asterics/Asterics-AAC) (grid.asterics.eu) — a free, offline-capable AAC/communication-board app. This repo does **not** fork the app; it builds container images from the upstream source and deploys them, plus the optional backend services the app can use (account sync, chat).

Full design rationale: see [`docs/deployment-plan.md`](docs/deployment-plan.md).

## What's in here

``` sh
docker/
  frontend/     Dockerfile + nginx.conf — clones & builds Asterics-AAC, serves static PWA
  couchauth/    Dockerfile for the couch-auth (superlogin) account/sync service
k8s/base/       Kubernetes manifests (kustomize base)
.github/workflows/  CI to build & push both images on push to main
docs/           Deployment plan this repo implements
```

## Components

| Component | Required? | What it does |
| --- | --- | --- |
| `aac-frontend` | Always | Static Vue PWA — the app itself |
| `aac-couchauth` | Only if you want account login / cross-device sync | Node/Express service (couch-auth), talks to CouchDB |
| `couchdb` | Only alongside couch-auth | Stores per-user grid data, one DB per user |
| Matrix homeserver | Only if you want the built-in chat feature | Not included here — point the app at an existing homeserver, or deploy `matrix-org/synapse`'s own Helm chart separately |

If you only need the standalone app with no account sync, deploy just `aac-frontend` and skip couchdb/couchauth.

## Quick start

1. Build & push images (or let CI do it — see `.github/workflows/build-and-push.yaml`):

   ```sh
   docker build -t <registry>/asterics-aac-frontend:latest -f docker/frontend/Dockerfile .
   docker build -t <registry>/asterics-aac-couchauth:latest -f docker/couchauth/Dockerfile .
   docker push <registry>/asterics-aac-frontend:latest
   docker push <registry>/asterics-aac-couchauth:latest
   ```

2. Copy `k8s/base/secrets.example.yaml` to `k8s/base/secrets.yaml`, fill in real values, and **do not commit it** (already gitignored). For production, use Sealed Secrets / External Secrets Operator / your cloud KMS instead of plain manifests.
3. Update image references in `k8s/base/frontend-deployment.yaml` and `k8s/base/couchauth-deployment.yaml` to point at your registry, and the host in `k8s/base/ingress.yaml`.
4. Deploy:

   ``` sh
   kubectl apply -k k8s/base
   ```

5. Watch rollout:

   ``` sh
   kubectl -n asterics-aac get pods -w
   ```

## Deploy order

CouchDB → couch-auth → frontend → Ingress. `kubectl apply -k` applies everything at once; Kubernetes will retry couch-auth/frontend pods until CouchDB is reachable, but for a clean first deploy it's worth applying in that order manually.

## Status

Scaffold generated — images have not yet been built/pushed and nothing has been deployed to a live cluster. Fill in your registry, domain, and secrets before running `kubectl apply`.
