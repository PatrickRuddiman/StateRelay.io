# 09 - Event System & Real-time Updates

## Overview

VirtDevice uses Redis Pub/Sub for real-time event propagation across services. When a device state changes, all interested services are notified immediately.

## Event Flow

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          EVENT FLOW                                      │
└─────────────────────────────────────────────────────────────────────────┘

  State Change Source          Redis Pub/Sub              Subscribers
  ──────────────────          ──────────────              ───────────

  ┌──────────────┐
  │  Alexa Cmd   │──┐
  └──────────────┘  │
                    │        ┌──────────────┐        ┌──────────────┐
  ┌──────────────┐  ├───────▶│    Redis     │───────▶│ IFTTT Notify │
  │ IFTTT Action │──┤        │   Pub/Sub    │        │  (Realtime)  │
  └──────────────┘  │        │              │        └──────────────┘
                    │        │  Channel:    │
  ┌──────────────┐  │        │ device:*:    │        ┌──────────────┐
  │   Web API    │──┤        │   state      │───────▶│    Alexa     │
  └──────────────┘  │        │              │        │ ChangeReport │
                    │        └──────────────┘        └──────────────┘
  ┌──────────────┐  │              │
  │  Webhooks    │──┘              │                 ┌──────────────┐
  └──────────────┘                 └────────────────▶│  WebSocket   │
                                                     │  (Dashboard) │
                                                     └──────────────┘
```

## Redis Pub/Sub Channels

| Channel Pattern | Purpose |
|-----------------|---------|
| `device:{deviceId}:state` | Device property changes |
| `user:{userId}:events` | User-level events |
| `global:events` | System-wide events |

## Event Payload

```typescript
interface DeviceStateEvent {
  type: 'property_changed';
  deviceId: string;
  userId: string;
  tenantId: string;
  property: string;
  oldValue: string | null;
  newValue: string;
  source: 'alexa' | 'ifttt' | 'web' | 'api';
  timestamp: string;
}
```

## Publisher Implementation

```typescript
// services/api/src/services/event-publisher.ts
import Redis from 'ioredis';

const redis = new Redis(process.env.REDIS_URL);

export async function publishStateChange(event: DeviceStateEvent) {
  const channel = `device:${event.deviceId}:state`;
  await redis.publish(channel, JSON.stringify(event));
  
  // Also publish to user channel for dashboard updates
  await redis.publish(`user:${event.userId}:events`, JSON.stringify(event));
}
```

## Subscriber Implementation

```typescript
// services/alexa-skill/src/subscribers/state-subscriber.ts
import Redis from 'ioredis';

const subscriber = new Redis(process.env.REDIS_URL);

subscriber.psubscribe('device:*:state');

subscriber.on('pmessage', async (pattern, channel, message) => {
  const event: DeviceStateEvent = JSON.parse(message);
  
  // Don't send ChangeReport if Alexa caused the change
  if (event.source === 'alexa') return;
  
  await sendChangeReport(event.userId, event.deviceId, event.property, event.newValue);
});
```

## PostgreSQL NOTIFY Integration

Database triggers can also publish events:

```sql
CREATE OR REPLACE FUNCTION notify_property_change()
RETURNS TRIGGER AS $$
BEGIN
  PERFORM pg_notify('property_changes', json_build_object(
    'device_id', NEW.device_id,
    'property', NEW.name,
    'old_value', OLD.value,
    'new_value', NEW.value
  )::text);
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

## WebSocket for Dashboard

```typescript
// services/web/src/lib/websocket.ts
import { Server } from 'socket.io';
import Redis from 'ioredis';

const subscriber = new Redis(process.env.REDIS_URL);

export function setupWebSocket(io: Server) {
  io.on('connection', (socket) => {
    const userId = socket.handshake.auth.userId;
    
    // Subscribe to user events
    const channel = `user:${userId}:events`;
    subscriber.subscribe(channel);
    
    subscriber.on('message', (ch, message) => {
      if (ch === channel) {
        socket.emit('device:update', JSON.parse(message));
      }
    });
  });
}
```

## Activity Logging

All events are also logged for audit:

```typescript
export async function logActivity(event: DeviceStateEvent) {
  await prisma.activityLog.create({
    data: {
      tenantId: event.tenantId,
      userId: event.userId,
      deviceId: event.deviceId,
      action: 'property_changed',
      source: event.source,
      details: {
        property: event.property,
        oldValue: event.oldValue,
        newValue: event.newValue,
      },
    },
  });
}
