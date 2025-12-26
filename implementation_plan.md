# Implementation Plan

[Overview]
Build the StateRelay multi-tenant SaaS platform for virtual smart-home devices that bridge Amazon Alexa and IFTTT ecosystems.

StateRelay is a comprehensive platform operated by Pinnacle Ridge Cloud Solutions LLC that creates virtual "Relay Devices" (switches, dimmers, sensors) appearing as real devices to both Amazon Alexa and IFTTT. The architecture consists of 8 Docker containers: Caddy (reverse proxy), Web (Next.js dashboard), API (Express REST), Auth (OAuth 2.0 server), Alexa Skill (Smart Home handler), IFTTT Service (IFTTT protocol), PostgreSQL, and Redis. All containers are built via GitHub Actions and pushed to GHCR. The system implements multi-tenant isolation via PostgreSQL Row-Level Security, Stripe billing for subscriptions, and real-time event propagation via Redis Pub/Sub. The domain is staterelay.io with subdomains for each service.

[Types]
Define all TypeScript enums, interfaces, and database types based on architecture documents 04, 08, and 10.

### Database Enums (from architecture/04-database-design.md)
- `user_role`: owner, admin, member
- `device_type`: switch, dimmer, sensor
- `property_data_type`: boolean, integer, float, string
- `subscription_tier`: free, pro, business
- `subscription_status`: active, past_due, canceled, trialing
- `oauth_client_type`: web, alexa, ifttt, mobile
- `activity_action`: device_created, device_updated, device_deleted, property_changed, user_login, user_logout, subscription_created, subscription_updated, alexa_command, ifttt_trigger, ifttt_action

### Core Interfaces (from architecture/08-device-types.md)
- Device: id, tenantId, userId, name, friendlyName, deviceType, description, manufacturer, model, alexaEndpointId, alexaDisplayCategories, alexaCapabilities, iftttDeviceId, isOnline, lastStateChange, metadata, timestamps
- DeviceProperty: id, deviceId, name, value, dataType, minValue, maxValue, allowedValues, unit, alexaInterface, alexaPropertyName, lastChangedBy, lastChangedAt, timestamps
- User: id, tenantId, email, passwordHash, name, role, emailVerified, tokens, lastLoginAt, settings, timestamps
- Subscription: id, userId, stripeCustomerId, stripeSubscriptionId, tier, status, maxDevices, features, currentPeriod, timestamps
- OAuthClient: id, clientId, clientSecret, name, type, redirectUris, allowedScopes, tokenLifetimes, timestamps
- OAuthToken: id, userId, clientId, accessToken, refreshToken, expirations, scope, timestamps
- AlexaLwaToken: id, userId, tokens, expirations, region, timestamps
- IftttTriggerSubscription: id, userId, triggerType, triggerFields, triggerIdentity, iftttSource, enabled, timestamps
- Webhook: id, userId, deviceId, url, events, secret, headers, enabled, metrics, timestamps
- ActivityLog: id, tenantId, userId, deviceId, action, details, source, ipAddress, userAgent, createdAt

### Event Types (from architecture/09-event-system.md)
- DeviceStateEvent: type, deviceId, userId, tenantId, property, oldValue, newValue, source, timestamp

[Files]
Create complete project structure with 8 service directories, shared packages, and configuration files.

### Root Level Files
- `/docker-compose.yml` - Orchestration for all 8 services
- `/docker-compose.dev.yml` - Development overrides
- `/.env.example` - Environment variable template
- `/.github/workflows/build.yml` - CI/CD pipeline for GHCR
- `/.github/dependabot.yml` - Dependency updates (already exists)
- `/package.json` - Workspace root for monorepo
- `/turbo.json` - Turborepo configuration (optional)
- `/README.md` - Updated project readme

### Service Directories
1. `/services/caddy/` - Reverse proxy
   - `Dockerfile`
   - `Caddyfile`

2. `/services/postgres/` - Database
   - `Dockerfile`
   - `init.sql` - Schema, enums, RLS policies, triggers

3. `/services/redis/` - Cache/PubSub
   - `Dockerfile`

4. `/services/auth/` - OAuth 2.0 Server
   - `Dockerfile`
   - `package.json`
   - `tsconfig.json`
   - `src/index.ts`
   - `src/config/oauth.ts`
   - `src/controllers/authorize.ts`
   - `src/controllers/token.ts`
   - `src/controllers/userinfo.ts`
   - `src/controllers/revoke.ts`
   - `src/middleware/authenticate.ts`
   - `src/middleware/validate-client.ts`
   - `src/services/token-service.ts`
   - `src/services/user-service.ts`
   - `src/views/login.ejs`
   - `src/views/consent.ejs`

5. `/services/api/` - REST API
   - `Dockerfile`
   - `package.json`
   - `tsconfig.json`
   - `prisma/schema.prisma`
   - `prisma/migrations/` - Generated
   - `src/index.ts`
   - `src/config/database.ts`
   - `src/controllers/device-controller.ts`
   - `src/controllers/property-controller.ts`
   - `src/controllers/user-controller.ts`
   - `src/controllers/webhook-controller.ts`
   - `src/controllers/subscription-controller.ts`
   - `src/middleware/authenticate.ts`
   - `src/middleware/rate-limit.ts`
   - `src/middleware/validate.ts`
   - `src/services/device-service.ts`
   - `src/services/event-publisher.ts`
   - `src/services/subscription-service.ts`
   - `src/services/stripe-service.ts`

6. `/services/alexa-skill/` - Alexa Smart Home
   - `Dockerfile`
   - `package.json`
   - `tsconfig.json`
   - `src/index.ts`
   - `src/handlers/discovery.ts`
   - `src/handlers/power-controller.ts`
   - `src/handlers/brightness-controller.ts`
   - `src/handlers/report-state.ts`
   - `src/handlers/accept-grant.ts`
   - `src/services/change-report.ts`
   - `src/services/lwa-token-service.ts`
   - `src/subscribers/state-subscriber.ts`

7. `/services/ifttt-service/` - IFTTT Protocol
   - `Dockerfile`
   - `package.json`
   - `tsconfig.json`
   - `src/index.ts`
   - `src/controllers/status.ts`
   - `src/controllers/user-info.ts`
   - `src/controllers/test-setup.ts`
   - `src/triggers/power-changed.ts`
   - `src/triggers/brightness-changed.ts`
   - `src/triggers/sensor-value-changed.ts`
   - `src/actions/set-power.ts`
   - `src/actions/set-brightness.ts`
   - `src/actions/set-sensor-value.ts`
   - `src/services/realtime.ts`
   - `src/subscribers/state-subscriber.ts`

8. `/services/web/` - Next.js Dashboard
   - `Dockerfile`
   - `package.json`
   - `next.config.js`
   - `tailwind.config.js`
   - `tsconfig.json`
   - `src/app/layout.tsx`
   - `src/app/page.tsx`
   - `src/app/login/page.tsx`
   - `src/app/register/page.tsx`
   - `src/app/dashboard/page.tsx`
   - `src/app/dashboard/devices/page.tsx`
   - `src/app/dashboard/devices/new/page.tsx`
   - `src/app/dashboard/devices/[id]/page.tsx`
   - `src/app/dashboard/billing/page.tsx`
   - `src/app/dashboard/settings/page.tsx`
   - `src/app/oauth/authorize/page.tsx`
   - `src/components/` - UI components
   - `src/lib/api.ts`
   - `src/lib/auth.ts`
   - `src/lib/websocket.ts`

### Shared Packages
- `/packages/shared/` - Shared types and utilities
  - `package.json`
  - `src/types/device.ts`
  - `src/types/user.ts`
  - `src/types/events.ts`
  - `src/constants/device-templates.ts`

[Functions]
No existing functions to modify - this is a greenfield project. All functions listed in Files section are new.

### Key Functions by Service
**Auth Service**: authorizeHandler, tokenHandler, userinfoHandler, revokeHandler, generateAccessToken, generateRefreshToken, verifyAccessToken, createTokenPair, refreshTokens, verifyPassword, hashPassword

**API Service**: createDevice, getDevice, updateDevice, deleteDevice, updateProperty, publishStateChange, createWebhook, createCheckoutSession, handleStripeWebhook, canCreateDevice

**Alexa Skill**: handleDiscovery, handlePowerController, handleBrightnessController, handleReportState, handleAcceptGrant, sendChangeReport, getValidLwaToken, refreshLwaToken

**IFTTT Service**: powerChangedTrigger, brightnessChangedTrigger, sensorValueChangedTrigger, setPowerAction, setBrightnessAction, setSensorValueAction, notifyIfttt, deviceOptions

**Web App**: useDevices, useDevice, useAuth, useWebSocket, DeviceCard, DeviceForm, PropertyEditor, BillingPage

[Classes]
No existing classes to modify - this is a greenfield project. Primary patterns use functional approach with services.

### Service Classes
- `TokenService` (auth) - JWT generation, validation, refresh
- `DeviceService` (api) - Device CRUD operations
- `EventPublisher` (api) - Redis pub/sub publishing
- `StripeService` (api) - Payment processing
- `ChangeReportService` (alexa) - Alexa Event Gateway
- `LwaTokenService` (alexa) - Login with Amazon tokens
- `RealtimeService` (ifttt) - IFTTT Realtime API

[Dependencies]
Install all required npm packages for each service and configure Prisma ORM.

### Root Workspace
- turbo (optional - monorepo build)

### Auth Service
- express, express-session, jsonwebtoken, bcrypt, ejs, ioredis, prisma, @prisma/client, cors, helmet, morgan, zod

### API Service
- express, prisma, @prisma/client, ioredis, jsonwebtoken, stripe, cors, helmet, morgan, express-rate-limit, zod, uuid

### Alexa Skill Service
- express, prisma, @prisma/client, ioredis, jsonwebtoken, axios, uuid, cors

### IFTTT Service
- express, prisma, @prisma/client, ioredis, jsonwebtoken, axios, cors

### Web Dashboard
- next, react, react-dom, tailwindcss, @tailwindcss/forms, socket.io-client, axios, lucide-react, zod, react-hook-form

### Shared
- typescript, @types/node, tsx (for all Node services)

[Testing]
Implement unit tests, integration tests, and end-to-end tests across all phases.

### Phase 1 Tests
- Docker container health checks
- Database connection tests
- Redis connection tests
- Schema migration validation

### Phase 2 Tests
- OAuth flow integration tests
- Token generation/validation unit tests
- API endpoint unit tests
- Device CRUD integration tests
- Property update integration tests
- Event publishing tests

### Phase 3 Tests
- Alexa Discovery handler tests
- Alexa directive handler tests
- ChangeReport sending tests (mocked)
- AcceptGrant flow tests

### Phase 4 Tests
- IFTTT trigger endpoint tests
- IFTTT action endpoint tests
- Realtime API notification tests (mocked)
- Dynamic field options tests

### Phase 5 Tests
- Next.js page rendering tests
- Authentication flow tests
- Device management UI tests
- WebSocket connection tests

### Phase 6 Tests
- End-to-end Alexa → IFTTT flow
- End-to-end IFTTT → Alexa flow
- Stripe checkout flow tests
- Rate limiting tests
- Security header tests

[Implementation Order]
Execute implementation in 6 phases matching the project timeline from architecture/01-overview.md.

## Phase 1: Foundation (2 weeks)
Infrastructure setup, Docker configuration, database schema, and CI/CD pipeline.

### 1.1 Project Structure Setup
- Create monorepo directory structure (/services/*)
- Create root package.json with workspaces
- Create .env.example with all environment variables
- Update root README.md with project overview

### 1.2 Docker Infrastructure
- Create Dockerfile for each service (8 total):
  - services/caddy/Dockerfile (FROM caddy:2-alpine)
  - services/postgres/Dockerfile (FROM postgres:16-alpine)
  - services/redis/Dockerfile (FROM redis:7-alpine)
  - services/api/Dockerfile (FROM node:20-alpine)
  - services/auth/Dockerfile (FROM node:20-alpine)
  - services/alexa-skill/Dockerfile (FROM node:20-alpine)
  - services/ifttt-service/Dockerfile (FROM node:20-alpine)
  - services/web/Dockerfile (FROM node:20-alpine, multi-stage)
- Create docker-compose.yml with all services
- Create docker-compose.dev.yml for local development
- Create Caddyfile with subdomain routing

### 1.3 GitHub Actions CI/CD
- Create .github/workflows/build.yml
- Configure matrix build for all 8 services
- Set up GHCR push with tagging (latest + SHA)
- Generate and configure GitHub secrets via gh CLI:
  - JWT_SECRET, SESSION_SECRET
  - POSTGRES_PASSWORD, REDIS_PASSWORD
  - Placeholder secrets for external services

### 1.4 Database Setup
- Create services/postgres/init.sql with:
  - PostgreSQL extensions (uuid-ossp, pgcrypto)
  - All enum types (7 enums)
  - All tables (11 tables)
  - All indexes
  - Row-Level Security policies
  - Database triggers (updated_at, notify, device_limit, ID generation)
  - OAuth client seed data
- Create Prisma schema in services/api/prisma/schema.prisma
- Generate initial Prisma migration

### 1.5 Redis Configuration
- Configure Redis with password authentication
- Set up persistence (RDB snapshots)

### 1.6 Shared Package
- Create packages/shared/ structure
- Define TypeScript types for all entities
- Define device templates constants
- Configure package.json for workspace linking

---

## Phase 2: Core API & Auth (2 weeks)
OAuth 2.0 server implementation and REST API with device management.

### 2.1 Auth Service - OAuth Server
- Initialize services/auth/ with Express
- Implement OAuth 2.0 authorization code flow:
  - GET /oauth/authorize - Authorization endpoint
  - POST /oauth/token - Token endpoint (auth_code + refresh)
  - GET /oauth/userinfo - User info endpoint
  - POST /oauth/revoke - Token revocation
- Create login.ejs and consent.ejs views
- Implement TokenService (JWT generation, validation)
- Implement user authentication (bcrypt)
- Configure session management with Redis
- Implement CSRF protection (state parameter)

### 2.2 Auth Service - User Registration
- POST /register - User registration
- Email verification flow (optional for MVP)
- Password reset flow (optional for MVP)

### 2.3 API Service - Core Setup
- Initialize services/api/ with Express
- Configure Prisma client
- Configure Redis client
- Implement authentication middleware (JWT validation)
- Implement rate limiting middleware
- Implement request validation middleware (Zod)
- Set up error handling and logging

### 2.4 API Service - Device Endpoints
- GET /v1/devices - List devices
- POST /v1/devices - Create device (with template properties)
- GET /v1/devices/:id - Get device
- PATCH /v1/devices/:id - Update device
- DELETE /v1/devices/:id - Delete device
- Implement device limit enforcement

### 2.5 API Service - Property Endpoints
- PATCH /v1/devices/:id/properties/:name - Update property
- Implement event publishing on property change
- Log activity on property changes

### 2.6 API Service - User Endpoints
- GET /v1/users/me - Get current user
- PATCH /v1/users/me - Update profile

### 2.7 API Service - Webhook Endpoints
- GET /v1/webhooks - List webhooks
- POST /v1/webhooks - Create webhook
- DELETE /v1/webhooks/:id - Delete webhook
- Implement webhook dispatch on events

### 2.8 Event System
- Implement EventPublisher service
- Configure Redis pub/sub channels:
  - device:{deviceId}:state
  - user:{userId}:events
- Bridge PostgreSQL NOTIFY to Redis
- Implement activity logging

---

## Phase 3: Alexa Integration (2 weeks)
Smart Home Skill with device discovery, control, and proactive state reports.

### 3.1 Alexa Skill - Core Setup
- Initialize services/alexa-skill/ with Express
- Configure authentication (JWT from OAuth)
- Set up Redis subscriber for state changes
- Configure Prisma client

### 3.2 Alexa Skill - Discovery
- Implement Alexa.Discovery.Discover handler
- Build endpoint list with capabilities
- Map device types to Alexa display categories
- Build capability list per device type

### 3.3 Alexa Skill - Control Handlers
- Implement Alexa.PowerController (TurnOn, TurnOff)
- Implement Alexa.BrightnessController (Set, Adjust)
- Implement Alexa.ReportState
- Update device properties on commands
- Publish events to Redis after commands

### 3.4 Alexa Skill - AcceptGrant
- Implement Alexa.Authorization.AcceptGrant
- Exchange LWA authorization code for tokens
- Store LWA tokens in alexa_lwa_tokens table
- Handle token refresh

### 3.5 Alexa Skill - ChangeReport
- Implement ChangeReportService
- Subscribe to Redis state change events
- Send proactive ChangeReport to Alexa Event Gateway
- Filter out Alexa-sourced changes (avoid loops)
- Implement LWA token refresh when expired

### 3.6 Alexa Developer Console Setup (Manual)
- Document steps to create Alexa Skill
- Configure Smart Home API
- Set up account linking with OAuth endpoints
- Add skill permissions (alexa::async_event:write)
- Configure endpoint URL (alexa.staterelay.io)

---

## Phase 4: IFTTT Integration (2 weeks)
IFTTT Service with triggers, actions, and realtime notifications.

### 4.1 IFTTT Service - Core Setup
- Initialize services/ifttt-service/ with Express
- Configure authentication (Bearer token + IFTTT-Service-Key)
- Set up Redis subscriber
- Configure Prisma client

### 4.2 IFTTT Service - Status & User
- GET /ifttt/v1/status - Service health check
- GET /ifttt/v1/user/info - Return user info
- POST /ifttt/v1/test/setup - Test environment setup

### 4.3 IFTTT Service - Triggers
- POST /ifttt/v1/triggers/power_changed
- POST /ifttt/v1/triggers/brightness_changed
- POST /ifttt/v1/triggers/sensor_value_changed
- Implement trigger field dynamic options:
  - /triggers/*/fields/device_id/options
- Store trigger subscriptions in database
- Return recent events from activity_logs

### 4.4 IFTTT Service - Actions
- POST /ifttt/v1/actions/set_power
- POST /ifttt/v1/actions/set_brightness
- POST /ifttt/v1/actions/set_sensor_value
- Update device properties
- Publish events to Redis

### 4.5 IFTTT Service - Realtime API
- Implement RealtimeService
- Subscribe to Redis state change events
- Send notifications to IFTTT Realtime API
- Match trigger subscriptions to events
- Filter out IFTTT-sourced changes (avoid loops)

### 4.6 IFTTT Platform Setup (Manual)
- Document steps to apply for IFTTT Platform access
- Configure service OAuth with our endpoints
- Define triggers and actions in IFTTT console
- Set service key environment variable

---

## Phase 5: Frontend Dashboard (2 weeks)
Next.js web application with device management and billing.

### 5.1 Web App - Project Setup
- Initialize services/web/ with Next.js 14
- Configure Tailwind CSS with brand colors
- Install and configure Lucide icons
- Set up TypeScript configuration
- Create layout.tsx with navigation

### 5.2 Web App - Authentication
- Create /login page
- Create /register page
- Implement auth context/provider
- Store tokens in httpOnly cookies (via API)
- Implement protected route wrapper

### 5.3 Web App - Dashboard
- Create /dashboard page with overview
- Show device count, subscription tier
- Recent activity feed

### 5.4 Web App - Device Management
- Create /dashboard/devices page (list)
- Create /dashboard/devices/new page (create form)
- Create /dashboard/devices/[id] page (edit/view)
- Implement DeviceCard component
- Implement DeviceForm component
- Implement PropertyEditor component
- Real-time updates via WebSocket

### 5.5 Web App - WebSocket Integration
- Implement Socket.io client connection
- Subscribe to user events channel
- Update device states in real-time
- Show connection status indicator

### 5.6 Web App - Billing
- Create /dashboard/billing page
- Show current subscription tier
- Upgrade/downgrade buttons
- Integrate Stripe Checkout redirect
- Integrate Stripe Customer Portal redirect

### 5.7 Web App - Settings
- Create /dashboard/settings page
- Profile editing
- Password change (optional for MVP)
- Connected services status (Alexa, IFTTT)

### 5.8 Web App - OAuth Consent
- Create /oauth/authorize page
- Show consent screen for third-party OAuth
- Handle approve/deny actions

---

## Phase 6: Polish & Launch (2 weeks)
Testing, security hardening, documentation, and platform submissions.

### 6.1 Stripe Integration
- Create Stripe products and prices
- Implement checkout session creation
- Implement webhook handler for:
  - checkout.session.completed
  - customer.subscription.updated
  - customer.subscription.deleted
  - invoice.payment_failed
- Implement customer portal session creation
- Enforce device limits based on subscription

### 6.2 Security Hardening
- Verify all security headers in Caddy
- Test rate limiting on all endpoints
- Verify CORS configuration
- Test input validation on all endpoints
- Verify RLS policies work correctly
- Test token expiration and refresh
- Verify secrets not in logs

### 6.3 Testing
- Write unit tests for core services
- Write integration tests for API
- Write integration tests for OAuth flow
- Test Alexa directive handlers
- Test IFTTT trigger/action endpoints
- End-to-end flow testing
- Load testing (optional)

### 6.4 Documentation
- Update README.md with setup instructions
- Create docs/ folder or docs.staterelay.io content
- Document API endpoints
- Document OAuth flow for integrators
- Create troubleshooting guide

### 6.5 Platform Submissions
- Submit Alexa Skill for certification
- Submit IFTTT Service for review
- Address any certification feedback
- Publish approved integrations

### 6.6 Monitoring Setup (Optional)
- Configure logging to stdout (JSON)
- Set up health check endpoints
- Configure uptime monitoring
- Set up error alerting

### 6.7 Launch Preparation
- Final security review
- Final load testing
- Create backup strategy
- Configure DNS for staterelay.io
- Deploy to production
- Monitor initial users
