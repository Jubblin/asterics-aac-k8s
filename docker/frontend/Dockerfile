# Builds the AsTeRICS AAC static PWA from upstream source and serves it via nginx.
#
# Build:
#   docker build -t asterics-aac-frontend:latest -f docker/frontend/Dockerfile .
#
# Override the upstream ref if you want to pin to a release tag instead of master:
#   docker build --build-arg ASTERICS_REF=v<tag> ...

ARG ASTERICS_REPO=https://github.com/asterics/Asterics-AAC.git
ARG ASTERICS_REF=master

FROM node:26-alpine@sha256:6bd9ab43c0ec0190ca7b2d193906cb8b7edf6e7f4afa43ddeccd6452710bffb5 AS build
ARG ASTERICS_REPO
ARG ASTERICS_REF
RUN apk add --no-cache git python3 make g++ \
    && corepack enable
WORKDIR /src
RUN git clone --depth 1 --branch "${ASTERICS_REF}" "${ASTERICS_REPO}" . \
    && yarn install --frozen-lockfile \
# npm run build runs: jest && webpack --env production && getServiceWorkerCachePaths.js
    && npm run build

FROM nginx:1.31-alpine@sha256:fed7e14be83cbd2661a75cdc5b872d936cf155d46c327084d30b8df19ae88d72
COPY --from=build /src/app/build /usr/share/nginx/html
COPY docker/frontend/nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
HEALTHCHECK --interval=30s --timeout=3s CMD wget -qO- http://localhost/ || exit 1
