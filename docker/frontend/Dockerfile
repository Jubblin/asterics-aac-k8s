# Builds the AsTeRICS AAC static PWA from upstream source and serves it via nginx.
#
# Build:
#   docker build -t asterics-aac-frontend:latest -f docker/frontend/Dockerfile .
#
# Override the upstream ref if you want to pin to a release tag instead of master:
#   docker build --build-arg ASTERICS_REF=v<tag> ...

ARG ASTERICS_REPO=https://github.com/asterics/Asterics-AAC.git
ARG ASTERICS_REF=master

FROM node:24-alpine@sha256:ebfe2f90462722a7a4de65e91990e97fe0d401c70e0e762c5b53302f905ec1c1 AS build
ARG ASTERICS_REPO
ARG ASTERICS_REF
RUN apk add --no-cache git python3 make g++ \
    && corepack enable
WORKDIR /src
RUN git clone --depth 1 --branch "${ASTERICS_REF}" "${ASTERICS_REPO}" . \
    && yarn install --frozen-lockfile \
# npm run build runs: jest && webpack --env production && getServiceWorkerCachePaths.js
    && npm run build

FROM nginx:1.31-alpine@sha256:1ed1b0e1d7652937d6cbdaf4018c7b6fc009a7dd6c3047351e2eddda745de43f
COPY --from=build /src/app/build /usr/share/nginx/html
COPY docker/frontend/nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
HEALTHCHECK --interval=30s --timeout=3s CMD wget -qO- http://localhost/ || exit 1
