# syntax=docker/dockerfile:1.7
FROM node:22-bookworm-slim AS base
ENV PNPM_HOME=/pnpm PATH=/pnpm:$PATH
RUN corepack enable
WORKDIR /app
FROM base AS build
RUN corepack prepare pnpm@11.13.1 --activate
COPY package.json pnpm-lock.yaml pnpm-workspace.yaml ./
COPY patches ./patches
COPY packages ./packages
COPY apps/backend ./apps/backend
COPY apps/frontend/package.json ./apps/frontend/package.json
COPY apps/mcp-server/package.json ./apps/mcp-server/package.json
COPY apps/browser-extension/package.json ./apps/browser-extension/package.json
RUN pnpm install --frozen-lockfile
RUN pnpm run build:shared-packages
RUN pnpm --filter @apps/backend build
FROM node:22-bookworm-slim AS runtime
ENV NODE_ENV=production TZ=Europe/Berlin
WORKDIR /app
RUN groupadd --system --gid 10001 app && useradd --system --uid 10001 --gid app --home-dir /app app
COPY --from=build --chown=app:app /app/apps/backend/build ./apps/backend/build
COPY --from=build --chown=app:app /app/apps/backend/src/core/db/migrations ./apps/backend/src/core/db/migrations
COPY --from=build --chown=app:app /app/apps/backend/package.json ./apps/backend/package.json
COPY --from=build --chown=app:app /app/node_modules ./node_modules
COPY --from=build --chown=app:app /app/apps/backend/node_modules ./apps/backend/node_modules
COPY --from=build --chown=app:app /app/packages ./packages
COPY --from=build --chown=app:app /app/pnpm-workspace.yaml ./pnpm-workspace.yaml
USER app
WORKDIR /app/apps/backend
EXPOSE 5001
CMD ["node", "build/src/index.js"]
