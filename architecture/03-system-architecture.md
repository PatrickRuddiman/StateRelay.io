# 03 - System Architecture

## High-Level Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                                    INTERNET                                              │
│                                                                                          │
│    ┌──────────┐     ┌──────────┐     ┌──────────┐     ┌──────────┐     ┌──────────┐   │
│    │  Users   │     │  Alexa   │     │  IFTTT   │     │  Stripe  │     │  LWA     │   │
│    │ Browser  │     │  Cloud   │     │  Cloud   │     │  API     │     │  OAuth   │   │
│    └────┬─────┘     └────┬─────┘     └────┬─────┘     └────┬─────┘     └────┬─────┘   │
│         │                │                │                │                │          │
└─────────┼────────────────┼────────────────┼────────────────┼────────────────┼──────────┘
          │                │                │                │                │
          ▼                ▼                ▼                ▼                ▼
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                                 CADDY REVERSE PROXY                                      │
│                                                                                          │
│  ┌─────────────────────────────────────────────────────────────────────────────────┐   │
│  │  TLS Termination (Auto HTTPS)  │  Rate Limiting  │  Request Routing  │  CORS    │   │
│  └─────────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                          │
│  Routes:                                                                                 │
│  ├── app.staterelay.io      → web:3000                                                  │
│  ├── api.staterelay.io      → api:4000                                                  │
│  ├── auth.staterelay.io     → auth:4003                                                 │
│  ├── alexa.staterelay.io    → alexa-skill:4001                                          │
│  └── ifttt.staterelay.io    → ifttt-service:4002                                        │
└────────────────────────────────────────────┬────────────────────────────────────────────┘
                                             │
                     ┌───────────────────────┼───────────────────────┐
                     │                       │                       │
                     ▼                       ▼                       ▼
┌────────────────────────────┐ ┌────────────────────────────┐ ┌────────────────────────────┐
│         WEB APP            │ │         API SERVICE        │ │      AUTH SERVICE          │
│        (Next.js)           │ │        (Express.js)        │ │      (Express.js)          │
│                            │ │                            │ │                            │
│  ┌──────────────────────┐  │ │  ┌──────────────────────┐  │ │  ┌──────────────────────┐  │
│  │  Dashboard           │  │ │  │  Device Controller   │  │ │  │  OAuth 2.0 Server    │  │
│  │  - Device Management │  │ │  │  - CRUD Operations   │  │ │  │  - Authorization     │  │
│  │  - Property Editing  │  │ │  │  - State Management  │  │ │  │  - Token Exchange    │  │
│  │  - Activity Logs     │  │ │  │  - Webhooks          │  │ │  │  - Token Refresh     │  │
│  │                      │  │ │  │                      │  │ │  │                      │  │
│  │  Auth UI             │  │ │  │  User Controller     │  │ │  │  User Info Endpoint  │  │
│  │  - Login/Register    │  │ │  │  - Profile           │  │ │  │  - /user/info        │  │
│  │  - OAuth Consent     │  │ │  │  - Preferences       │  │ │  │                      │  │
│  │                      │  │ │  │                      │  │ │  │  Session Management  │  │
│  │  Billing UI          │  │ │  │  Subscription Ctrl   │  │ │  │  - JWT Tokens        │  │
│  │  - Plans             │  │ │  │  - Stripe Webhooks   │  │ │  │  - Refresh Tokens    │  │
│  │  - Payment Methods   │  │ │  │  - Usage Tracking    │  │ │  │                      │  │
│  └──────────────────────┘  │ │  └──────────────────────┘  │ │  └──────────────────────┘  │
│                            │ │                            │ │                            │
│  Port: 3000                │ │  Port: 4000                │ │  Port: 4003                │
└────────────────────────────┘ └────────────┬───────────────┘ └────────────────────────────┘
                                            │
         ┌──────────────────────────────────┼──────────────────────────────────┐
         │                                  │                                  │
         ▼                                  │                                  ▼
┌────────────────────────────┐              │              ┌────────────────────────────┐
│     ALEXA SKILL SERVICE    │              │              │    IFTTT SERVICE           │
│        (Express.js)        │              │              │      (Express.js)          │
│                            │              │              │                            │
│  ┌──────────────────────┐  │              │              │  ┌──────────────────────┐  │
│  │  Discovery Handler   │  │              │              │  │  Trigger Endpoints   │  │
│  │  - Alexa.Discovery   │  │              │              │  │  - State Changed     │  │
│  │  - Device Listing    │  │              │              │  │  - Value Threshold   │  │
│  │                      │  │              │              │  │                      │  │
│  │  Directive Handlers  │  │              │              │  │  Action Endpoints    │  │
│  │  - PowerController   │  │              │              │  │  - Set Power         │  │
│  │  - BrightnessCtrl    │  │              │              │  │  - Set Brightness    │  │
│  │  - ReportState       │  │              │              │  │  - Set Sensor Value  │  │
│  │                      │  │              │              │  │                      │  │
│  │  AcceptGrant Handler │  │              │              │  │  Status Endpoint     │  │
│  │  - LWA Token Storage │  │              │              │  │  - /ifttt/v1/status  │  │
│  │                      │  │              │              │  │                      │  │
│  │  ChangeReport Sender │  │              │              │  │  User Info Endpoint  │  │
│  │  - Event Gateway     │  │              │              │  │  - /ifttt/v1/user    │  │
│  │  - Proactive Events  │  │              │              │  │                      │  │
│  └──────────────────────┘  │              │              │  │  Test Setup Endpoint │  │
│                            │              │              │  │  - /ifttt/v1/test    │  │
│  Port: 4001                │              │              │  └──────────────────────┘  │
└────────────────────────────┘              │              │                            │
         │                                  │              │  Port: 4002                │
         │                                  │              └────────────────────────────┘
         │                                  │                          │
         │                                  ▼                          │
         │              ┌─────────────────────────────────────┐        │
         │              │          CORE SERVICE LAYER         │        │
         │              │                                     │        │
         │              │  ┌───────────────────────────────┐  │        │
         │              │  │      Device Service           │  │        │
         │              │  │  - Create/Update/Delete       │  │        │
         │              │  │  - Property Management        │  │        │
         │              │  │  - State Persistence          │  │        │
         │              │  └───────────────────────────────┘  │        │
         │              │                                     │        │
         │              │  ┌───────────────────────────────┐  │        │
         │              │  │      Event Service            │  │        │
         │              │  │  - State Change Detection     │  │        │
         │              │  │  - Webhook Dispatch           │  │        │
         │              │  │  - Platform Notifications     │  │        │
         │              │  └───────────────────────────────┘  │        │
         │              │                                     │        │
         │              │  ┌───────────────────────────────┐  │        │
         │              │  │   Subscription Service        │  │        │
         │              │  │  - Alexa Event Gateway Subs   │  │        │
         │              │  │  - IFTTT Trigger Identity     │  │        │
         │              │  │  - Webhook Registrations      │  │        │
         │              │  └───────────────────────────────┘  │        │
         │              └─────────────────┬───────────────────┘        │
         │                                │                            │
         └────────────────────────────────┼────────────────────────────┘
                                          │
                      ┌───────────────────┴───────────────────┐
                      │                                       │
                      ▼                                       ▼
┌─────────────────────────────────────────┐ ┌─────────────────────────────────────────┐
│              POSTGRESQL                  │ │                REDIS                    │
│                                          │ │                                         │
│  ┌────────────────────────────────────┐  │ │  ┌─────────────────────────────────┐   │
│  │  Tables (with Row-Level Security)  │  │ │  │  Pub/Sub Channels               │   │
│  │  - tenants                         │  │ │  │  - device:state:{deviceId}      │   │
│  │  - users                           │  │ │  │  - user:events:{userId}         │   │
│  │  - devices                         │  │ │  └─────────────────────────────────┘   │
│  │  - device_properties               │  │ │                                         │
│  │  - oauth_clients                   │  │ │  ┌─────────────────────────────────┐   │
│  │  - oauth_tokens                    │  │ │  │  Session Cache                  │   │
│  │  - alexa_lwa_tokens                │  │ │  │  - session:{sessionId}          │   │
│  │  - ifttt_trigger_subscriptions     │  │ │  │  - user:{userId}:devices        │   │
│  │  - subscriptions                   │  │ │  └─────────────────────────────────┘   │
│  │  - webhooks                        │  │ │                                         │
│  │  - activity_logs                   │  │ │  ┌─────────────────────────────────┐   │
│  └────────────────────────────────────┘  │ │  │  Rate Limiting                  │   │
│                                          │ │  │  - ratelimit:{ip}               │   │
│  Port: 5432                              │ │  │  - ratelimit:{userId}           │   │
└─────────────────────────────────────────┘ │  └─────────────────────────────────┘   │
                                            │                                         │
                                            │  Port: 6379                             │
                                            └─────────────────────────────────────────┘
```

## Service Descriptions

### 1. Caddy Reverse Proxy

**Purpose**: Entry point for all HTTP/HTTPS traffic

**Responsibilities**:
- Automatic TLS certificate provisioning via Let's Encrypt
- Route requests to appropriate backend services
- Rate limiting to prevent abuse
- CORS header management
- Request/response logging

**Technology**: Caddy 2.x

**Configuration**: See [11-docker-deployment.md](./11-docker-deployment.md)

### 2. Web Application (Next.js)

**Purpose**: User-facing dashboard for device management

**Responsibilities**:
- User registration and login
- Device creation/editing/deletion
- Property value manipulation
- Activity log viewing
- Subscription management and billing

**Technology**: Next.js 14, React 18, Tailwind CSS

**Key Pages**:
```
/                     # Landing page
/login                # User login
/register             # User registration
/dashboard            # Main dashboard
/dashboard/devices    # Device list
/dashboard/devices/new   # Create device
/dashboard/devices/:id   # Edit device
/dashboard/billing    # Subscription management
/dashboard/settings   # User settings
/oauth/authorize      # OAuth consent screen
```

### 3. API Service (Express.js)

**Purpose**: Core REST API for all data operations

**Responsibilities**:
- Device CRUD operations
- Property state management
- User profile management
- Webhook management
- Subscription tracking

**Technology**: Node.js 20, Express.js, Prisma ORM

**Key Endpoints**: See [10-api-reference.md](./10-api-reference.md)

### 4. Auth Service (Express.js)

**Purpose**: OAuth 2.0 authorization server

**Responsibilities**:
- OAuth 2.0 authorization code flow
- Access token issuance
- Refresh token management
- User authentication
- Multi-client support (Alexa, IFTTT, Web)

**Technology**: Node.js 20, Express.js, node-oauth2-server

**Key Endpoints**:
```
GET  /oauth/authorize    # Authorization endpoint
POST /oauth/token        # Token endpoint
GET  /oauth/userinfo     # User info endpoint
POST /oauth/revoke       # Token revocation
```

### 5. Alexa Skill Service (Express.js)

**Purpose**: Handle Alexa Smart Home Skill directives

**Responsibilities**:
- Device discovery (Alexa.Discovery)
- Power control (Alexa.PowerController)
- Brightness control (Alexa.BrightnessController)
- State reporting (Alexa.ReportState)
- Proactive state updates (ChangeReport)

**Technology**: Node.js 20, Express.js

**Key Handlers**:
```javascript
// Discovery
Alexa.Discovery.Discover → Return all user devices

// Control
Alexa.PowerController.TurnOn → Set power = ON
Alexa.PowerController.TurnOff → Set power = OFF
Alexa.BrightnessController.SetBrightness → Set brightness = value
Alexa.BrightnessController.AdjustBrightness → Increment/decrement brightness

// Reporting
Alexa.ReportState → Return current device state
Alexa.Authorization.AcceptGrant → Store LWA tokens
```

### 6. IFTTT Service (Express.js)

**Purpose**: Implement IFTTT service protocol

**Responsibilities**:
- Trigger endpoint implementation
- Action endpoint implementation
- Realtime API notifications
- Dynamic field options
- Test setup endpoint

**Technology**: Node.js 20, Express.js

**Key Endpoints**:
```
GET  /ifttt/v1/status                    # Service status
GET  /ifttt/v1/user/info                 # User info
POST /ifttt/v1/test/setup                # Test setup

# Triggers
POST /ifttt/v1/triggers/power_changed    # Power state changed
POST /ifttt/v1/triggers/brightness_changed # Brightness changed
POST /ifttt/v1/triggers/sensor_value_changed # Sensor value changed

# Actions  
POST /ifttt/v1/actions/set_power         # Set power state
POST /ifttt/v1/actions/set_brightness    # Set brightness
POST /ifttt/v1/actions/set_sensor_value  # Set sensor value
```

### 7. PostgreSQL Database

**Purpose**: Persistent data storage

**Responsibilities**:
- User and tenant data
- Device and property storage
- OAuth tokens
- Subscription records
- Activity logging

**Technology**: PostgreSQL 16

**Multi-tenancy**: Row-Level Security (RLS) with `tenant_id` on all tables

### 8. Redis Cache

**Purpose**: Fast caching and pub/sub messaging

**Responsibilities**:
- Session storage
- Device state caching
- Pub/sub for real-time events
- Rate limiting counters

**Technology**: Redis 7

---

## Service Communication Patterns

### Synchronous Communication (HTTP)

```
┌──────────┐        ┌──────────┐        ┌──────────┐
│  Alexa   │───────▶│  Alexa   │───────▶│   API    │
│  Cloud   │  HTTPS │  Skill   │  HTTP  │ Service  │
└──────────┘        └──────────┘        └──────────┘
```

All inter-service communication within the Docker network uses HTTP. External communication uses HTTPS through Caddy.

### Asynchronous Communication (Redis Pub/Sub)

```
┌──────────┐        ┌──────────┐        ┌──────────┐
│   API    │───────▶│  Redis   │───────▶│  Alexa   │
│ Service  │ PUBLISH│  Pub/Sub │SUBSCRIBE│  Skill   │
└──────────┘        └──────────┘        └──────────┘
                          │
                          ├───────────▶┌──────────┐
                          │ SUBSCRIBE  │  IFTTT   │
                          │            │ Service  │
                          │            └──────────┘
                          │
                          └───────────▶┌──────────┐
                            SUBSCRIBE  │   Web    │
                                       │   App    │
                                       └──────────┘
```

When a device state changes:
1. API Service publishes event to Redis
2. Alexa Skill Service subscribes and sends ChangeReport
3. IFTTT Service subscribes and sends Realtime API notification
4. Web App subscribes and updates UI via WebSocket

---

## Request Flow Examples

### Flow 1: Alexa Voice Command

```
User: "Alexa, turn on the living room switch"

┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐
│  User   │───▶│  Alexa  │───▶│  Caddy  │───▶│ Alexa   │───▶│   API   │
│         │    │  Cloud  │    │         │    │ Skill   │    │ Service │
└─────────┘    └─────────┘    └─────────┘    └─────────┘    └─────────┘
                                                                 │
                                                                 ▼
                                                           ┌─────────┐
                                                           │Postgres │
                                                           │(update) │
                                                           └─────────┘
                                                                 │
                                                                 ▼
                                                           ┌─────────┐
                                                           │  Redis  │
                                                           │(publish)│
                                                           └─────────┘
                                                                 │
                              ┌──────────────────────────────────┤
                              │                                  │
                              ▼                                  ▼
                        ┌─────────┐                        ┌─────────┐
                        │  IFTTT  │                        │   Web   │
                        │ Service │                        │   App   │
                        │(notify) │                        │(update) │
                        └─────────┘                        └─────────┘
                              │
                              ▼
                        ┌─────────┐
                        │  IFTTT  │
                        │  Cloud  │
                        │Realtime │
                        │   API   │
                        └─────────┘
```

### Flow 2: IFTTT Applet Action

```
IFTTT: Action "Set brightness to 75%"

┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐
│  IFTTT  │───▶│  Caddy  │───▶│  IFTTT  │───▶│   API   │
│  Cloud  │    │         │    │ Service │    │ Service │
└─────────┘    └─────────┘    └─────────┘    └─────────┘
                                                   │
                                                   ▼
                                             ┌─────────┐
                                             │Postgres │
                                             │(update) │
                                             └─────────┘
                                                   │
                                                   ▼
                                             ┌─────────┐
                                             │  Redis  │
                                             │(publish)│
                                             └─────────┘
                                                   │
                              ┌────────────────────┤
                              │                    │
                              ▼                    ▼
                        ┌─────────┐          ┌─────────┐
                        │  Alexa  │          │   Web   │
                        │  Skill  │          │   App   │
                        │(Change  │          │(update) │
                        │ Report) │          └─────────┘
                        └─────────┘
                              │
                              ▼
                        ┌─────────┐
                        │  Alexa  │
                        │  Event  │
                        │ Gateway │
                        └─────────┘
                              │
                              ▼
                        ┌─────────┐
                        │  Alexa  │
                        │ Routine │
                        │(trigger)│
                        └─────────┘
```

### Flow 3: Web Dashboard Update

```
User: Changes dimmer to 50% in web dashboard

┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐
│  User   │───▶│   Web   │───▶│  Caddy  │───▶│   API   │
│Browser  │    │   App   │    │         │    │ Service │
└─────────┘    └─────────┘    └─────────┘    └─────────┘
                                                   │
                                                   ▼
                                             ┌─────────┐
                                             │Postgres │
                                             │(update) │
                                             └─────────┘
                                                   │
                                                   ▼
                                             ┌─────────┐
                                             │  Redis  │
                                             │(publish)│
                                             └─────────┘
                                                   │
                    ┌──────────────────────────────┼──────────────────────────────┐
                    │                              │                              │
                    ▼                              ▼                              ▼
              ┌─────────┐                    ┌─────────┐                    ┌─────────┐
              │  Alexa  │                    │  IFTTT  │                    │   Web   │
              │  Skill  │                    │ Service │                    │   App   │
              │(Change  │                    │(Realtime│                    │  (WS)   │
              │ Report) │                    │   API)  │                    └─────────┘
              └─────────┘                    └─────────┘
                    │                              │
                    ▼                              ▼
              ┌─────────┐                    ┌─────────┐
              │  Alexa  │                    │  IFTTT  │
              │  Cloud  │                    │  Cloud  │
              └─────────┘                    └─────────┘
```

---

## Scaling Considerations

### Horizontal Scaling

Each service can be scaled independently:

```yaml
# docker-compose.prod.yml
services:
  api:
    deploy:
      replicas: 3
  
  alexa-skill:
    deploy:
      replicas: 2
  
  ifttt-service:
    deploy:
      replicas: 2
```

### Database Scaling

For high traffic scenarios:
- Read replicas for query distribution
- Connection pooling with PgBouncer
- Consider partitioning by tenant_id for very large datasets

### Redis Scaling

- Redis Cluster for horizontal scaling
- Redis Sentinel for high availability

---

## Monitoring Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         Monitoring Stack                         │
│                                                                  │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐      │
│  │  Prometheus  │    │   Grafana    │    │   Loki       │      │
│  │  (metrics)   │    │ (dashboards) │    │  (logs)      │      │
│  └──────────────┘    └──────────────┘    └──────────────┘      │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
                              ▲
                              │ scrape/push
                              │
┌─────────────────────────────┴─────────────────────────────────┐
│                        All Services                            │
│  Each service exposes:                                         │
│  - /metrics endpoint (Prometheus format)                       │
│  - JSON logs to stdout (collected by Loki)                     │
└───────────────────────────────────────────────────────────────┘
```

### Key Metrics to Monitor

| Metric | Description | Alert Threshold |
|--------|-------------|-----------------|
| `http_requests_total` | Total HTTP requests | N/A (informational) |
| `http_request_duration_seconds` | Request latency | P95 > 2s |
| `device_state_changes_total` | State change events | N/A (informational) |
| `alexa_changereport_failures` | Failed ChangeReports | > 1% failure rate |
| `ifttt_realtime_notifications` | IFTTT notifications sent | N/A (informational) |
| `oauth_token_errors` | Token validation failures | > 10/minute |
| `db_connection_pool_usage` | DB pool utilization | > 80% |
| `redis_connected_clients` | Redis connections | > 100 |

---

## Disaster Recovery

### Backup Strategy

| Component | Backup Frequency | Retention | Method |
|-----------|------------------|-----------|--------|
| PostgreSQL | Every 6 hours | 30 days | pg_dump to S3 |
| Redis | Every hour | 7 days | RDB snapshots |
| Logs | Real-time | 90 days | Loki retention |

### Recovery Time Objectives (RTO)

| Scenario | RTO Target |
|----------|------------|
| Single service failure | < 1 minute (auto-restart) |
| Database failure | < 15 minutes |
| Complete infrastructure failure | < 1 hour |

### Recovery Point Objectives (RPO)

| Component | RPO Target |
|-----------|------------|
| Device state | 0 (synchronous replication) |
| User data | < 6 hours |
| Activity logs | < 1 hour |
