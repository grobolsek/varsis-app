FROM caddy:2.11.9-builder AS builder
RUN xcaddy build --with github.com/corazawaf/coraza-caddy/v2

FROM caddy:2.11.9-alpine
COPY --from=builder /usr/bin/caddy /usr/bin/caddy
