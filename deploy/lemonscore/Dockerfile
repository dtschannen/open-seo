# OpenSEO image for the LemonScore server (2 vCPU / 3.8 GB, shared with the Lead Engine).
#
# Differs from deploy/docker/Dockerfile in two ways:
# - The vite build runs here, in CI, instead of at every container start. The
#   SSR build wants a 4 GB heap, which that server cannot spare next to the
#   Lead Engine. The entrypoint skips its build when dist/.openseo-build-env
#   matches the runtime env, so AUTH_MODE and VITE_SHOW_DEVTOOLS below must stay
#   equal to what compose.yaml passes (and no other VITE_* value may be set).
# - It runs as the unprivileged `node` user instead of root.

# Use the full Node image so workerd has a working CA trust store for outbound HTTPS.
FROM node:22

# pnpm comes from corepack, prepared as root into a shared, read-only location.
ENV COREPACK_HOME=/opt/corepack
RUN corepack enable && corepack prepare pnpm@10.30.1 --activate && chmod -R a+rX /opt/corepack

RUN mkdir /app && chown node:node /app
WORKDIR /app
USER node

COPY --chown=node:node package.json pnpm-lock.yaml pnpm-workspace.yaml ./
COPY --chown=node:node patches ./patches
RUN pnpm install --frozen-lockfile

COPY --chown=node:node . .

# The build-relevant env, baked into the bundle. Keep in sync with compose.yaml.
ENV AUTH_MODE=local_noauth \
    VITE_SHOW_DEVTOOLS=false

# Same fingerprint as deploy/docker/docker-entrypoint.sh computes at start.
RUN pnpm run build \
 && env | grep -E '^(VITE_|AUTH_MODE|BYPASS_EMAIL_VERIFICATION|POSTHOG_PUBLIC_KEY|POSTHOG_HOST|TURNSTILE_SITE_KEY|POSTHOG_SOURCEMAPS)' | sort | sha256sum | cut -d' ' -f1 | tr -d '\n' > dist/.openseo-build-env \
 && mkdir -p /app/.wrangler

EXPOSE 3001

HEALTHCHECK --interval=30s --timeout=10s --start-period=120s --retries=3 \
  CMD node -e "fetch('http://127.0.0.1:'+(process.env.PORT||3001)+'/api/health').then(function(r){process.exit(r.ok?0:1)}).catch(function(){process.exit(1)})"

CMD ["sh", "deploy/docker/docker-entrypoint.sh"]
