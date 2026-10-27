# Multi-stage Dockerfile for Umbrel Bitcoin Node app
#
# * dev: `docker compose up` builds the **dev** stage only
# hot-reloading backend container (the UI dev server lives in its own
# apps/ui/Dockerfile; both share a hoisted node_modules volume in the compose file).
#
# * production: `docker build ...` produces a single self-contained image with backend + UI + bitcoind
# single final image that contains
#  – compiled backend
#  – bundled React UI (Vite output → dist/public)
#  – compiled shared libs (libs/settings, libs/shared-types)
#  – bitcoind + bitcoin-cli binaries

###########################################
# Bitcoind: the signed Plumb release for the target platform
###########################################
# Runs on the build machine's own platform and picks the tarball for the target one, so the arm64
# image needs no emulation here. The keyring must hold exactly the pinned key, SHA256SUMS must match
# its pinned hash and carry a good signature from that key, and the tarball must match SHA256SUMS.
FROM --platform=$BUILDPLATFORM debian:bookworm-slim AS plumb
ARG TARGETARCH
# One Plumb release. Change these two together, and the version in libs/settings/settings.meta.ts.
ARG PLUMB_VERSION=29.4.3.knots20260508.plumb8
ARG PLUMB_SHA256SUMS=0df3b788650b53658b6fd0570b2302e6887ccae72e3fc01fc0d7ddfa73dc395b
ENV PLUMB_SIGNER=89F0E41D72CE523F4AA1CDB692CDFFB7C40CD1BA
RUN apt-get update && apt-get install -y --no-install-recommends ca-certificates curl gnupg \
 && rm -rf /var/lib/apt/lists/*
COPY plumb/keys/ /tmp/plumb-keys/
WORKDIR /tmp/plumb
RUN set -e; \
    export GNUPGHOME="$(mktemp -d)" LC_ALL=C; \
    case "${TARGETARCH}" in amd64) ARCH=x86_64 ;; arm64) ARCH=aarch64 ;; *) echo "Unsupported TARGETARCH ${TARGETARCH}"; exit 1 ;; esac; \
    gpg --quiet --import /tmp/plumb-keys/*.asc; \
    if [ "$(gpg --with-colons --list-keys | grep -c '^pub:')" != 1 ] || \
       [ "$(gpg --with-colons --list-keys | awk -F: '/^fpr:/{print $10; exit}')" != "${PLUMB_SIGNER}" ]; then \
        echo "KEYRING IS NOT EXACTLY THE PINNED PLUMB KEY"; exit 1; \
    fi; \
    TARBALL="bitcoin-${PLUMB_VERSION}-${ARCH}-linux-gnu.tar.gz"; \
    BASE="https://github.com/plumb-node/plumb/releases/download/v${PLUMB_VERSION}"; \
    curl -fsSL --fail-early -o SHA256SUMS "${BASE}/SHA256SUMS" -o SHA256SUMS.asc "${BASE}/SHA256SUMS.asc" -o "${TARBALL}" "${BASE}/${TARBALL}"; \
    echo "${PLUMB_SHA256SUMS}  SHA256SUMS" | sha256sum -c -; \
    gpg --status-fd 1 --verify SHA256SUMS.asc SHA256SUMS > verify.txt 2>/dev/null || true; \
    if ! grep -q '^\[GNUPG:\] GOODSIG ' verify.txt || \
       ! grep -q "^\[GNUPG:\] VALIDSIG .* ${PLUMB_SIGNER}\$" verify.txt; then \
        cat verify.txt; echo "SHA256SUMS IS NOT SIGNED BY THE PINNED PLUMB KEY"; exit 1; \
    fi; \
    grep " ${TARBALL}\$" SHA256SUMS | sha256sum -c -; \
    mkdir -p /out; \
    tar -xzf "${TARBALL}" -C /out --strip-components=2 "bitcoin-${PLUMB_VERSION}/bin/bitcoind" "bitcoin-${PLUMB_VERSION}/bin/bitcoin-cli"; \
    echo "Plumb ${PLUMB_VERSION} ${ARCH}: SHA256SUMS signed by ${PLUMB_SIGNER}"; \
    sha256sum /out/bitcoind /out/bitcoin-cli

##########################################################
# Dependencies layer — install every workspace deps
# Functions as a shared cache for dev & app-builder stages
##########################################################
FROM node:20-slim AS deps
WORKDIR /repo

# Copy only dependency manifests to keep this layer cache-friendly
COPY package*.json ./
COPY apps/backend/package*.json apps/backend/
COPY apps/ui/package*.json apps/ui/

RUN npm ci --workspaces --include-workspace-root --install-strategy=hoisted

# Add npm binaries to PATH so subsequent stages can invoke tsc / vite / etc.
ENV PATH="/repo/node_modules/.bin:$PATH"

####################################################
# DEV-ONLY build stage: Backend hot-reload image
# - Not referenced by later COPY --from=… steps,
#   so it’s skipped during a plain `docker build`.
# - `docker compose up` targets it via `target: dev`
#   in the docker-compose.yml file.
####################################################
FROM deps AS dev
WORKDIR /repo/apps/backend

VOLUME /data

# bitcoind binaries
RUN mkdir -p /opt/bitcoind/v29.4.3.knots20260508.plumb8
COPY --from=plumb /out/bitcoind    /opt/bitcoind/v29.4.3.knots20260508.plumb8/bitcoind
COPY --from=plumb /out/bitcoin-cli /opt/bitcoind/v29.4.3.knots20260508.plumb8/bitcoin-cli

# Give 1000:1000 write permisions to /opt/bitcoind (just directory) so we can flip the symbolic link while the container runs as 1000:1000
RUN chown 1000:1000 /opt/bitcoind

# Symbolic link for default bitcoind binary
# By default we point to the Plumb release
RUN ln -sfn /opt/bitcoind/v29.4.3.knots20260508.plumb8 /opt/bitcoind/current
RUN ln -sfn /opt/bitcoind/current/bitcoind    /usr/local/bin/bitcoind
RUN ln -sfn /opt/bitcoind/current/bitcoin-cli /usr/local/bin/bitcoin-cli

# Symlink for datadir so bitcoin-cli auto-finds the RPC cookie
# Allows exec'ing bitcoin-cli inside the container without specifying credentials
RUN ln -sfn /data/bitcoin /root/.bitcoin
RUN mkdir -p /home/node \
 && ln -sfn /data/bitcoin /home/node/.bitcoin \
 && chown -h 1000:1000 /home/node/.bitcoin

CMD ["npm","run","dev"]

#################################################
# app-builder stage
#################################################
FROM deps AS app-builder
WORKDIR /repo
COPY . .

# build shared libraries (libs/*/dist)
# build backend (ts → js) (apps/backend/dist)
# bundle UI (vite build) (apps/ui/dist)
RUN tsc -b libs/shared-types libs/settings apps/backend \
 && npm run --workspace apps/ui build

########################################
# runtime stage — final production image
########################################
FROM node:20-slim AS runtime
WORKDIR /app

# Install production dependencies only
# Note: import-map lives in package.json, so is also needed so Node can resolve "#settings/#types"
  COPY package*.json ./
  COPY apps/backend/package*.json apps/backend/
  RUN npm ci --omit=dev --install-strategy=hoisted --ignore-scripts

# Backend JS bundle
COPY --from=app-builder /repo/apps/backend/dist ./dist

# Pre-bundled React UI → backend serves this from /public/*
COPY --from=app-builder /repo/apps/ui/dist ./dist/public

# Compiled shared libs (referenced at runtime via import-map)
COPY --from=app-builder /repo/libs/settings/dist ./libs/settings/dist
COPY --from=app-builder /repo/libs/shared-types/dist ./libs/shared-types/dist

# bitcoind binaries
RUN mkdir -p /opt/bitcoind/v29.4.3.knots20260508.plumb8
COPY --from=plumb /out/bitcoind    /opt/bitcoind/v29.4.3.knots20260508.plumb8/bitcoind
COPY --from=plumb /out/bitcoin-cli /opt/bitcoind/v29.4.3.knots20260508.plumb8/bitcoin-cli

# Give 1000:1000 write permisions to /opt/bitcoind (just directory) so we can flip the symbolic link while the container runs as 1000:1000
RUN chown 1000:1000 /opt/bitcoind

# Symbolic link for default bitcoind binary
# By default we point to the Plumb release
RUN ln -sfn /opt/bitcoind/v29.4.3.knots20260508.plumb8 /opt/bitcoind/current
RUN ln -sfn /opt/bitcoind/current/bitcoind    /usr/local/bin/bitcoind
RUN ln -sfn /opt/bitcoind/current/bitcoin-cli /usr/local/bin/bitcoin-cli

# Symlink for datadir so bitcoin-cli auto-finds the RPC cookie
# Allows exec'ing bitcoin-cli inside the container without specifying credentials
RUN ln -sfn /data/bitcoin /root/.bitcoin
RUN mkdir -p /home/node \
 && ln -sfn /data/bitcoin /home/node/.bitcoin \
 && chown -h 1000:1000 /home/node/.bitcoin

 RUN apt-get update && apt-get install -y \
    curl


# The release binaries link everything but glibc in; fail the build if this base lacks a library.
RUN if ldd /opt/bitcoind/current/bitcoind /opt/bitcoind/current/bitcoin-cli | grep 'not found'; then exit 1; fi \
 && d="$(mktemp -d)" && /opt/bitcoind/current/bitcoind -datadir="$d" -version | grep -F '.plumb' && rm -rf "$d"

ENV NODE_ENV=production

# Data dirs inside /data
ENV BITCOIN_DIR=/data/bitcoin
ENV APP_STATE_DIR=/data/app

# 3000 = HTTP API, 8332 = RPC, 8333 = P2P
EXPOSE 3000 8332 8333
# app and bitcoind data
VOLUME /data

CMD ["node", "dist/server.js"]