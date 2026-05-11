FROM node:22-alpine AS base

# Pin pnpm version to v9 to ensure compatibility with SurfSense's build scripts
RUN corepack enable && corepack prepare pnpm@9.15.4 --activate

# Install dependencies only when needed
FROM base AS deps
RUN apk add --no-cache libc6-compat
WORKDIR /app

# Copy package files
COPY package.json pnpm-lock.yaml* .npmrc* ./

# First copy the config file and content to avoid fumadocs-mdx postinstall error
COPY source.config.ts ./
COPY content ./content

# Install dependencies with frozen lockfile (No flags needed for pnpm 9)
RUN pnpm i --frozen-lockfile

# Rebuild the source code only when needed
FROM base AS builder
WORKDIR /app

# Build with placeholder values for NEXT_PUBLIC_* variables.
ARG NEXT_PUBLIC_FASTAPI_BACKEND_URL=__NEXT_PUBLIC_FASTAPI_BACKEND_URL__
ARG NEXT_PUBLIC_FASTAPI_BACKEND_AUTH_TYPE=__NEXT_PUBLIC_FASTAPI_BACKEND_AUTH_TYPE__
ARG NEXT_PUBLIC_ETL_SERVICE=__NEXT_PUBLIC_ETL_SERVICE__
ARG NEXT_PUBLIC_ZERO_CACHE_URL=__NEXT_PUBLIC_ZERO_CACHE_URL__
ARG NEXT_PUBLIC_DEPLOYMENT_MODE=__NEXT_PUBLIC_DEPLOYMENT_MODE__

ENV NEXT_PUBLIC_FASTAPI_BACKEND_URL=$NEXT_PUBLIC_FASTAPI_BACKEND_URL
ENV NEXT_PUBLIC_FASTAPI_BACKEND_AUTH_TYPE=$NEXT_PUBLIC_FASTAPI_BACKEND_AUTH_TYPE
ENV NEXT_PUBLIC_ETL_SERVICE=$NEXT_PUBLIC_ETL_SERVICE
ENV NEXT_PUBLIC_ZERO_CACHE_URL=$NEXT_PUBLIC_ZERO_CACHE_URL
ENV NEXT_PUBLIC_DEPLOYMENT_MODE=$NEXT_PUBLIC_DEPLOYMENT_MODE

COPY --from=deps /app/node_modules ./node_modules
COPY . .

RUN pnpm run build

# Production image, copy all the files and run next
FROM base AS runner
WORKDIR /app

ENV NODE_ENV=production

RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 nextjs

COPY --from=builder /app/public ./public
COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone/app/ ./
COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static

COPY --chown=nextjs:nodejs docker-entrypoint.js ./docker-entrypoint.js
COPY --chown=nextjs:nodejs --chmod=755 docker-entrypoint.sh ./docker-entrypoint.sh

USER nextjs
EXPOSE 3000
ENV PORT=3000
ENV HOSTNAME="0.0.0.0"

ENTRYPOINT ["/bin/sh", "./docker-entrypoint.sh"]
