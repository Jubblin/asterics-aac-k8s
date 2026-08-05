# Builds the couch-auth (superlogin) service that Asterics-AAC uses for
# "online user" accounts and cross-device sync against CouchDB.
#
# Good news: upstream superlogin/start.js already reads its CouchDB and
# mailer config from environment variables via dotenv-flow — no source patch
# is required for basic operation. See docker/couchauth/README.md for the
# full list of supported env vars and a caveat about email sending.
#
# Build:
#   docker build -t asterics-aac-couchauth:latest -f docker/couchauth/Dockerfile .

ARG ASTERICS_REPO=https://github.com/asterics/Asterics-AAC.git
ARG ASTERICS_REF=master

FROM node:24-alpine
ARG ASTERICS_REPO
ARG ASTERICS_REF
RUN apk add --no-cache git
WORKDIR /app
RUN git clone --depth 1 --branch "${ASTERICS_REF}" "${ASTERICS_REPO}" .
RUN corepack enable && yarn install --frozen-lockfile --production \
    && yarn cache clean

EXPOSE 3000
CMD ["node", "superlogin/start.js"]
