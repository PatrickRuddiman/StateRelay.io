# 10 - REST API Reference

## Base URL

```
https://api.staterelay.io/v1
```

## Authentication

All endpoints require Bearer token authentication:
```
Authorization: Bearer {access_token}
```

---

## Devices

### List Devices
```http
GET /devices
```

**Response:**
```json
{
  "data": [
    {
      "id": "uuid",
      "name": "living-room-switch",
      "friendlyName": "Living Room Switch",
      "deviceType": "switch",
      "isOnline": true,
      "properties": [
        { "name": "power", "value": "ON", "dataType": "boolean" }
      ],
      "createdAt": "2025-01-15T10:00:00Z"
    }
  ]
}
```

### Create Device
```http
POST /devices
Content-Type: application/json

{
  "name": "bedroom-dimmer",
  "friendlyName": "Bedroom Dimmer",
  "deviceType": "dimmer"
}
```

### Get Device
```http
GET /devices/{deviceId}
```

### Update Device
```http
PATCH /devices/{deviceId}
Content-Type: application/json

{
  "friendlyName": "New Name"
}
```

### Delete Device
```http
DELETE /devices/{deviceId}
```

---

## Properties

### Update Property
```http
PATCH /devices/{deviceId}/properties/{propertyName}
Content-Type: application/json

{
  "value": "ON"
}
```

**Response:**
```json
{
  "name": "power",
  "value": "ON",
  "dataType": "boolean",
  "lastChangedBy": "api",
  "lastChangedAt": "2025-01-15T10:30:00Z"
}
```

---

## User

### Get Current User
```http
GET /users/me
```

**Response:**
```json
{
  "id": "uuid",
  "email": "user@example.com",
  "name": "John Doe",
  "subscription": {
    "tier": "pro",
    "maxDevices": 20,
    "status": "active"
  }
}
```

### Update Profile
```http
PATCH /users/me
Content-Type: application/json

{
  "name": "New Name"
}
```

---

## Webhooks

### List Webhooks
```http
GET /webhooks
```

### Create Webhook
```http
POST /webhooks
Content-Type: application/json

{
  "url": "https://example.com/hook",
  "events": ["property_changed"],
  "deviceId": "uuid-or-null"
}
```

### Delete Webhook
```http
DELETE /webhooks/{webhookId}
```

---

## Activity Logs

### List Activity
```http
GET /activity?limit=50&deviceId={optional}
```

---

## Error Responses

```json
{
  "error": {
    "code": "DEVICE_NOT_FOUND",
    "message": "Device not found",
    "details": { "deviceId": "abc123" }
  }
}
```

| Code | HTTP | Description |
|------|------|-------------|
| `INVALID_REQUEST` | 400 | Bad request |
| `INVALID_TOKEN` | 401 | Auth failed |
| `FORBIDDEN` | 403 | No permission |
| `DEVICE_NOT_FOUND` | 404 | Device missing |
| `DEVICE_LIMIT_REACHED` | 403 | Plan limit |
| `RATE_LIMITED` | 429 | Too many requests |
