# syntax=docker/dockerfile:1.7
# DeepSeek Harness team server: builds the monorepo and serves `dsh web`.
FROM node:24-bookworm
# rc.8+ stamps browser artifacts with the source commit; the docker context
# ships without .git, so pass it (40 zeros = unknown). Override with:
#   DSH_CLIENT_COMMIT_HASH=$(git rev-parse HEAD) docker compose build
ARG DSH_CLIENT_COMMIT_HASH=0000000000000000000000000000000000000000


# Toolchain for native modules (node-pty, landlock addon); pnpm pinned to the
# repo's packageManager field.
RUN apt-get update \
 && apt-get install -y --no-install-recommends git ca-certificates python3 make g++ bubblewrap musl-tools \
 && rm -rf /var/lib/apt/lists/* \
 && npm install -g pnpm@11.7.0 lavish-axi@0.1.62

WORKDIR /app
COPY . .

# Keep the pnpm store outside the image layer. A failed or repeated build can
# reuse completed downloads; lower concurrency avoids registry timeout storms.
RUN --mount=type=cache,id=dsh-pnpm-store,target=/pnpm/store,sharing=locked \
    pnpm config set store-dir /pnpm/store \
 && pnpm config set fetch-retries 8 \
 && pnpm config set fetch-timeout 300000 \
 && pnpm config set network-concurrency 4 \
 && pnpm fetch --frozen-lockfile --prefer-offline \
 && pnpm install --frozen-lockfile --offline \
 && pnpm --dir native/landlock-run run build:native \
 && pnpm run build

# Profiles auto-initialize here on first boot; sessions persist across restarts.
ENV DSH_HOME=/data/dsh
VOLUME /data/dsh
RUN mkdir -p /data/dsh /data/lavish /workspaces && chown -R node:node /app /workspaces /data

# Agent sessions use the invoking directory as their default workspace root.
WORKDIR /workspaces
VOLUME /workspaces

HEALTHCHECK --interval=30s --timeout=5s --start-period=20s \
  CMD node -e "fetch('http://127.0.0.1:3080').then(r=>process.exit(r.ok?0:1)).catch(()=>process.exit(1))"

ENTRYPOINT ["node", "--expose-internals", "/app/apps/cli/lib/bin.js"]
USER node
CMD ["web"]
