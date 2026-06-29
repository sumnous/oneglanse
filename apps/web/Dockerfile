# syntax=docker/dockerfile:1.7

FROM node:20-bookworm-slim AS base
WORKDIR /app
ARG PNPM_VERSION=10.16.0
RUN npm install -g pnpm@${PNPM_VERSION}

FROM base AS deps
COPY package.json pnpm-lock.yaml pnpm-workspace.yaml .npmrc turbo.json tsconfig.json ./
COPY apps/web ./apps/web
COPY packages/db ./packages/db
COPY packages/errors ./packages/errors
COPY packages/services ./packages/services
COPY packages/types ./packages/types
COPY packages/ui ./packages/ui
COPY packages/utils ./packages/utils

RUN --mount=type=cache,id=pnpm-store-web,target=/root/.local/share/pnpm/store \
    pnpm install --frozen-lockfile

FROM deps AS builder
ENV SKIP_ENV_VALIDATION=true
ENV DATABASE_URL=postgres://stub/stub
ENV BETTER_AUTH_SECRET=docker-build-secret
ENV INTERNAL_CRON_SECRET=docker-build-secret
RUN pnpm turbo build --filter=@oneglanse/web...

FROM node:20-alpine AS runner
WORKDIR /app
ENV NODE_ENV=production
ENV PORT=3000
ARG PNPM_VERSION=10.16.0

COPY --from=builder /app/apps/web/.next/standalone ./
COPY --from=builder /app/apps/web/.next/static ./apps/web/.next/static
COPY --from=builder /app/apps/web/public ./apps/web/public

RUN corepack enable && corepack prepare pnpm@${PNPM_VERSION} --activate

WORKDIR /workspace
COPY package.json pnpm-lock.yaml pnpm-workspace.yaml .npmrc ./
COPY packages/db ./packages/db
RUN --mount=type=cache,id=pnpm-store-web,target=/root/.local/share/pnpm/store \
    pnpm install --prod --frozen-lockfile --filter @oneglanse/db...

WORKDIR /app

RUN chown -R node:node /app /workspace
USER node

EXPOSE 3000

CMD ["node", "apps/web/server.js"]

# === Migration runner (extends runner stage) ===
FROM runner AS runtime
WORKDIR /workspace

# Copy drizzle config and migrations from builder
COPY --from=builder /app/packages/db/drizzle.config.ts ./packages/db/drizzle.config.ts
COPY --from=builder /app/packages/db/drizzle ./packages/db/drizzle
COPY --from=builder /app/packages/db/package.json ./packages/db/package.json
COPY --from=builder /app/node_modules/.pnpm ./node_modules/.pnpm

# Create entrypoint script that runs migration then starts server
RUN echo '#!/bin/sh' > /entrypoint.sh && \
    echo 'set -e' >> /entrypoint.sh && \
    echo 'echo "Running database migrations..."' >> /entrypoint.sh && \
    echo 'cd /workspace && npx drizzle-kit migrate --config packages/db/drizzle.config.ts || echo "Migration failed, continuing..."' >> /entrypoint.sh && \
    echo 'echo "Starting web server..."' >> /entrypoint.sh && \
    echo 'exec node apps/web/server.js' >> /entrypoint.sh && \
    chmod +x /entrypoint.sh

ENTRYPOINT ["/entrypoint.sh"]
