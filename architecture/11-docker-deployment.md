# 11 - Docker & Deployment

## Docker Compose Architecture

All services run in Docker containers orchestrated by Docker Compose.

## docker-compose.yml

```yaml
version: '3.8'

services:
  # Reverse Proxy
  caddy:
    build: ./services/caddy
    image: ghcr.io/patrickruddiman/staterelay-caddy:latest
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - caddy_data:/data
      - caddy_config:/config
    depends_on:
      - web
      - api
      - auth
      - alexa-skill
      - ifttt-service
    restart: unless-stopped

  # Web Dashboard (Next.js)
  web:
    build: ./services/web
    image: ghcr.io/patrickruddiman/staterelay-web:latest
    environment:
      - NEXT_PUBLIC_API_URL=${API_URL}
      - NEXT_PUBLIC_AUTH_URL=${AUTH_URL}
    depends_on:
      - api
    restart: unless-stopped

  # REST API
  api:
    build: ./services/api
    image: ghcr.io/patrickruddiman/staterelay-api:latest
    environment:
      - DATABASE_URL=${DATABASE_URL}
      - REDIS_URL=${REDIS_URL}
      - JWT_SECRET=${JWT_SECRET}
    depends_on:
      - postgres
      - redis
    restart: unless-stopped

  # OAuth Server
  auth:
    build: ./services/auth
    image: ghcr.io/patrickruddiman/staterelay-auth:latest
    environment:
      - DATABASE_URL=${DATABASE_URL}
      - JWT_SECRET=${JWT_SECRET}
      - SESSION_SECRET=${SESSION_SECRET}
    depends_on:
      - postgres
      - redis
    restart: unless-stopped

  # Alexa Smart Home Skill
  alexa-skill:
    build: ./services/alexa-skill
    image: ghcr.io/patrickruddiman/staterelay-alexa:latest
    environment:
      - DATABASE_URL=${DATABASE_URL}
      - REDIS_URL=${REDIS_URL}
      - JWT_SECRET=${JWT_SECRET}
      - LWA_CLIENT_ID=${LWA_CLIENT_ID}
      - LWA_CLIENT_SECRET=${LWA_CLIENT_SECRET}
    depends_on:
      - postgres
      - redis
    restart: unless-stopped

  # IFTTT Service
  ifttt-service:
    build: ./services/ifttt-service
    image: ghcr.io/patrickruddiman/staterelay-ifttt:latest
    environment:
      - DATABASE_URL=${DATABASE_URL}
      - REDIS_URL=${REDIS_URL}
      - JWT_SECRET=${JWT_SECRET}
      - IFTTT_SERVICE_KEY=${IFTTT_SERVICE_KEY}
    depends_on:
      - postgres
      - redis
    restart: unless-stopped

  # PostgreSQL Database
  postgres:
    build: ./services/postgres
    image: ghcr.io/patrickruddiman/staterelay-postgres:latest
    environment:
      - POSTGRES_USER=${POSTGRES_USER}
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
      - POSTGRES_DB=${POSTGRES_DB}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    restart: unless-stopped

  # Redis Cache
  redis:
    build: ./services/redis
    image: ghcr.io/patrickruddiman/staterelay-redis:latest
    command: redis-server --requirepass ${REDIS_PASSWORD}
    volumes:
      - redis_data:/data
    restart: unless-stopped

volumes:
  caddy_data:
  caddy_config:
  postgres_data:
  redis_data:
```

## Service Dockerfiles

### services/caddy/Dockerfile
```dockerfile
FROM caddy:2-alpine
COPY Caddyfile /etc/caddy/Caddyfile
```

### services/web/Dockerfile
```dockerfile
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:20-alpine
WORKDIR /app
COPY --from=builder /app/.next ./.next
COPY --from=builder /app/public ./public
COPY --from=builder /app/package*.json ./
COPY --from=builder /app/node_modules ./node_modules
EXPOSE 3000
CMD ["npm", "start"]
```

### services/api/Dockerfile
```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
RUN npx prisma generate
EXPOSE 4000
CMD ["node", "dist/index.js"]
```

### services/postgres/Dockerfile
```dockerfile
FROM postgres:16-alpine
COPY init.sql /docker-entrypoint-initdb.d/
```

### services/redis/Dockerfile
```dockerfile
FROM redis:7-alpine
```

## Caddyfile

```caddyfile
{
    email {$ACME_EMAIL}
}

app.{$DOMAIN} {
    reverse_proxy web:3000
}

api.{$DOMAIN} {
    reverse_proxy api:4000
}

auth.{$DOMAIN} {
    reverse_proxy auth:4003
}

alexa.{$DOMAIN} {
    reverse_proxy alexa-skill:4001
}

ifttt.{$DOMAIN} {
    reverse_proxy ifttt-service:4002
}
```

## GitHub Actions CI/CD

### .github/workflows/build.yml

```yaml
name: Build and Push Docker Images

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_PREFIX: ${{ github.repository_owner }}/staterelay

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    strategy:
      matrix:
        service:
          - caddy
          - web
          - api
          - auth
          - alexa-skill
          - ifttt-service
          - postgres
          - redis

    steps:
      - uses: actions/checkout@v4

      - name: Log in to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: ./services/${{ matrix.service }}
          push: ${{ github.event_name != 'pull_request' }}
          tags: |
            ${{ env.REGISTRY }}/${{ env.IMAGE_PREFIX }}-${{ matrix.service }}:latest
            ${{ env.REGISTRY }}/${{ env.IMAGE_PREFIX }}-${{ matrix.service }}:${{ github.sha }}
```

## Environment Variables (.env.example)

```bash
# Domain
DOMAIN=staterelay.io
ACME_EMAIL=admin@staterelay.io

# Database
DATABASE_URL=postgresql://staterelay:password@postgres:5432/staterelay
POSTGRES_USER=staterelay
POSTGRES_PASSWORD=change-me-in-production
POSTGRES_DB=staterelay

# Redis
REDIS_URL=redis://:password@redis:6379
REDIS_PASSWORD=change-me-in-production

# Auth
JWT_SECRET=generate-256-bit-secret
SESSION_SECRET=generate-another-secret

# Alexa
LWA_CLIENT_ID=amzn1.application-oa2-client.xxx
LWA_CLIENT_SECRET=your-secret

# IFTTT
IFTTT_SERVICE_KEY=your-service-key

# Stripe
STRIPE_SECRET_KEY=sk_live_xxx
STRIPE_WEBHOOK_SECRET=whsec_xxx
```

## GitHub Secrets Setup

Run these commands to set secrets via GitHub CLI:

```bash
# Generate secrets
JWT_SECRET=$(openssl rand -base64 32)
SESSION_SECRET=$(openssl rand -base64 32)
POSTGRES_PASSWORD=$(openssl rand -base64 24)
REDIS_PASSWORD=$(openssl rand -base64 24)

# Set secrets
gh secret set JWT_SECRET --body "$JWT_SECRET"
gh secret set SESSION_SECRET --body "$SESSION_SECRET"
gh secret set POSTGRES_PASSWORD --body "$POSTGRES_PASSWORD"
gh secret set REDIS_PASSWORD --body "$REDIS_PASSWORD"

# Set external service secrets (get these from dashboards)
gh secret set LWA_CLIENT_ID --body "your-client-id"
gh secret set LWA_CLIENT_SECRET --body "your-secret"
gh secret set IFTTT_SERVICE_KEY --body "your-key"
gh secret set STRIPE_SECRET_KEY --body "sk_live_xxx"
gh secret set STRIPE_WEBHOOK_SECRET --body "whsec_xxx"
```

## Deployment Commands

```bash
# Pull latest images
docker compose pull

# Start services
docker compose up -d

# View logs
docker compose logs -f

# Run migrations
docker compose exec api npx prisma migrate deploy

# Restart a service
docker compose restart api
