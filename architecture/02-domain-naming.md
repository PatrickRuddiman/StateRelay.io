# 02 - Domain & Naming Conventions

## Registered Domain

**Primary Domain**: `staterelay.io` ✅ (Registered)

**Operated by**: Pinnacle Ridge Cloud Solutions LLC

## Subdomain Structure

```
staterelay.io                    # Marketing website
├── app.staterelay.io            # User dashboard (Next.js)
├── api.staterelay.io            # REST API (public + internal)
├── auth.staterelay.io           # OAuth server / account linking
├── alexa.staterelay.io          # Alexa skill endpoint
├── ifttt.staterelay.io          # IFTTT service endpoint
├── docs.staterelay.io           # Documentation
└── status.staterelay.io         # Uptime & incidents
```

**Secondary Domain**: `staterelay.app` (optional - redirect to .io or reserve for mobile/PWA)

### Caddy Route Configuration

```caddyfile
# Primary domain - Marketing site
staterelay.io {
    root * /var/www/marketing
    file_server
}

# App dashboard
app.staterelay.io {
    reverse_proxy web:3000
}

# API
api.staterelay.io {
    reverse_proxy api:4000
}

# OAuth server
auth.staterelay.io {
    reverse_proxy auth:4003
}

# Alexa Smart Home Skill
alexa.staterelay.io {
    reverse_proxy alexa-skill:4001
}

# IFTTT Service
ifttt.staterelay.io {
    reverse_proxy ifttt-service:4002
}
```

---

## Naming Conventions

### Code Style

| Element | Convention | Example |
|---------|------------|---------|
| **Files** | kebab-case | `device-controller.js` |
| **Directories** | kebab-case | `src/alexa-skill/` |
| **Classes** | PascalCase | `DeviceController` |
| **Functions** | camelCase | `getDeviceById()` |
| **Constants** | SCREAMING_SNAKE_CASE | `MAX_DEVICES_FREE_TIER` |
| **Environment Variables** | SCREAMING_SNAKE_CASE | `POSTGRES_PASSWORD` |
| **Database Tables** | snake_case | `device_properties` |
| **Database Columns** | snake_case | `created_at` |
| **API Endpoints** | kebab-case | `GET /api/v1/relay-devices` |
| **Docker Services** | kebab-case | `alexa-skill` |
| **Docker Images** | kebab-case with prefix | `ghcr.io/username/staterelay-api` |

### API Versioning

```
https://api.staterelay.io/v1/devices
https://api.staterelay.io/v1/properties
https://api.staterelay.io/v2/devices  # Future version
```

### Resource Naming

| Resource | Singular | Plural | Endpoint |
|----------|----------|--------|----------|
| Device | device | devices | `/v1/devices` |
| Property | property | properties | `/v1/devices/:id/properties` |
| User | user | users | `/v1/users` |
| Subscription | subscription | subscriptions | `/v1/subscriptions` |
| Webhook | webhook | webhooks | `/v1/webhooks` |

### Docker Image Naming

```
ghcr.io/patrickruddiman/staterelay-web:latest
ghcr.io/patrickruddiman/staterelay-api:latest
ghcr.io/patrickruddiman/staterelay-auth:latest
ghcr.io/patrickruddiman/staterelay-alexa:latest
ghcr.io/patrickruddiman/staterelay-ifttt:latest
ghcr.io/patrickruddiman/staterelay-postgres:latest
ghcr.io/patrickruddiman/staterelay-redis:latest
ghcr.io/patrickruddiman/staterelay-caddy:latest
```

### Environment Variable Prefixes

| Prefix | Purpose | Example |
|--------|---------|---------|
| `SR_` | StateRelay app config | `SR_LOG_LEVEL=debug` |
| `DB_` | Database config | `DB_HOST=postgres` |
| `REDIS_` | Redis config | `REDIS_PASSWORD=secret` |
| `ALEXA_` | Alexa integration | `ALEXA_CLIENT_ID=xxx` |
| `IFTTT_` | IFTTT integration | `IFTTT_SERVICE_KEY=xxx` |
| `STRIPE_` | Payment config | `STRIPE_SECRET_KEY=xxx` |
| `JWT_` | Authentication | `JWT_SECRET=xxx` |

---

## OAuth Client Identifiers

### Internal OAuth Clients

| Client Name | Client ID | Purpose |
|-------------|-----------|---------|
| StateRelay Web | `staterelay-web` | Dashboard login |
| StateRelay Mobile | `staterelay-mobile` | Future mobile app |

### External OAuth Clients

| Client Name | Client ID | Purpose |
|-------------|-----------|---------|
| Amazon Alexa | `alexa-smart-home` | Alexa account linking |
| IFTTT | `ifttt-service` | IFTTT account linking |

---

## StateRelay Terminology

| Concept | StateRelay Term | Description |
|---------|-----------------|-------------|
| State Manager | StateRelay Core | Central state management |
| ChangeReport | State Relay Event | Proactive state notification |
| Virtual Device | Relay Device | Software-defined device |
| Property | Relay Property | Device attribute |
| Trigger | Relay Event | State change notification |
| Action | State Update | Property modification |

## Device Type Identifiers

| Device Type | Alexa Display Category | IFTTT Identifier |
|-------------|----------------------|------------------|
| Relay Switch | `SWITCH` | `relay_switch` |
| Relay Dimmer | `LIGHT` | `relay_dimmer` |
| Relay Sensor | `CONTACT_SENSOR` | `relay_sensor` |

### Device Naming in User Interface

Users will name their Relay Devices with friendly names like:
- "Living Room Switch"
- "Garage Door Sensor"
- "Bedroom Dimmer"

These names are used by:
- Alexa: "Alexa, turn on Living Room Switch"
- IFTTT: Displayed in dropdown menus

---

## Error Code Conventions

### HTTP Status Code Usage

| Code | Meaning | Usage |
|------|---------|-------|
| 200 | OK | Successful GET, PUT, PATCH |
| 201 | Created | Successful POST |
| 204 | No Content | Successful DELETE |
| 400 | Bad Request | Invalid input |
| 401 | Unauthorized | Missing/invalid token |
| 403 | Forbidden | Valid token, no permission |
| 404 | Not Found | Resource doesn't exist |
| 409 | Conflict | Duplicate resource |
| 422 | Unprocessable Entity | Validation error |
| 429 | Too Many Requests | Rate limited |
| 500 | Internal Server Error | Server error |

### Error Response Format

```json
{
  "error": {
    "code": "DEVICE_NOT_FOUND",
    "message": "Device with ID 'abc123' was not found",
    "details": {
      "deviceId": "abc123"
    }
  }
}
```

### Error Codes

| Code | HTTP Status | Description |
|------|-------------|-------------|
| `INVALID_REQUEST` | 400 | Malformed request body |
| `MISSING_FIELD` | 400 | Required field not provided |
| `INVALID_TOKEN` | 401 | Token expired or invalid |
| `TOKEN_EXPIRED` | 401 | Access token has expired |
| `FORBIDDEN` | 403 | User lacks permission |
| `DEVICE_NOT_FOUND` | 404 | Device ID doesn't exist |
| `USER_NOT_FOUND` | 404 | User ID doesn't exist |
| `DEVICE_LIMIT_REACHED` | 403 | Subscription device limit |
| `DUPLICATE_DEVICE` | 409 | Device name already exists |
| `RATE_LIMITED` | 429 | Too many requests |
| `INTERNAL_ERROR` | 500 | Unexpected server error |

---

## Logging Conventions

### Log Levels

| Level | Usage |
|-------|-------|
| `error` | Unrecoverable errors, exceptions |
| `warn` | Recoverable issues, deprecations |
| `info` | Normal operations, state changes |
| `debug` | Detailed debugging information |
| `trace` | Very detailed, performance-impacting |

### Log Format (JSON)

```json
{
  "timestamp": "2025-01-15T10:30:00.000Z",
  "level": "info",
  "service": "api",
  "message": "Device state updated",
  "context": {
    "deviceId": "abc123",
    "property": "power",
    "oldValue": "OFF",
    "newValue": "ON",
    "triggeredBy": "alexa"
  },
  "traceId": "trace-xyz-789"
}
```

### Trace ID Format

Used to track requests across services:
```
sr-{service}-{timestamp}-{random}
Example: sr-api-1705312200-a1b2c3
