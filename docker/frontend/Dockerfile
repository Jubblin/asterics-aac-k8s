# Builds the AsTeRICS AAC static PWA from upstream source and serves it via nginx.
#
# Build:
#   docker build -t asterics-aac-frontend:latest -f docker/frontend/Dockerfile .
#
# Override the upstream ref if you want to pin to a release tag instead of master:
#   docker build --build-arg ASTERICS_REF=v<tag> ...

ARG ASTERICS_REPO=https://github.com/asterics/Asterics-AAC.git
ARG ASTERICS_REF=master

FROM node:24-alpine@sha256:50c8e8ca1d27439048670df5883f32d57cf81cff6233222c893fd0d9884cbd81 AS build
ARG ASTERICS_REPO
ARG ASTERICS_REF
RUN apk add --no-cache git python3 make g++ \
    && corepack enable
WORKDIR /src
RUN git clone --depth 1 --branch "${ASTERICS_REF}" "${ASTERICS_REPO}" . \
    && yarn install --frozen-lockfile \
# npm run build runs: jest && webpack --env production && getServiceWorkerCachePaths.js
    && npm run build

FROM nginx:1.31-alpine@sha256:72ba65eb42c10344912a84ff42408db7d34f2feb642204570ab8fc5ffd29f1d3
COPY --from=build /src/app/build /usr/share/nginx/html
COPY docker/frontend/nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
HEALTHCHECK --interval=30s --timeout=3s CMD wget -qO- http://localhost/ || exit 1
