# 07 - IFTTT Service Integration

## Overview

VirtDevice integrates with IFTTT as a Service, enabling:
- **Triggers**: Fire when virtual device states change
- **Actions**: Set virtual device states from IFTTT applets
- **Realtime API**: Push state changes instantly to IFTTT

## IFTTT Service Endpoints

| Endpoint | Purpose |
|----------|---------|
| `GET /ifttt/v1/status` | Service health check |
| `GET /ifttt/v1/user/info` | Get authenticated user info |
| `POST /ifttt/v1/test/setup` | Test environment setup |
| `POST /ifttt/v1/triggers/{trigger_slug}` | Trigger polling/realtime |
| `POST /ifttt/v1/actions/{action_slug}` | Execute actions |

## Authentication

IFTTT uses OAuth 2.0 with our auth server. Every request includes:
```
Authorization: Bearer {user_access_token}
IFTTT-Service-Key: {service_key}
```

## Trigger Implementation

### power_changed Trigger

Fires when a device's power state changes.

```typescript
// POST /ifttt/v1/triggers/power_changed
export async function powerChangedTrigger(req: Request, res: Response) {
  const { trigger_identity, triggerFields, limit = 50 } = req.body;
  const { device_id } = triggerFields;
  
  // Get recent power state changes
  const events = await prisma.activityLog.findMany({
    where: {
      userId: req.user.id,
      deviceId: device_id || undefined,
      action: 'property_changed',
      details: { path: ['property'], equals: 'power' },
    },
    orderBy: { createdAt: 'desc' },
    take: limit,
  });
  
  const data = events.map(e => ({
    device_name: e.details.deviceName,
    power_state: e.details.newValue,
    changed_at: e.createdAt.toISOString(),
    meta: {
      id: e.id,
      timestamp: Math.floor(e.createdAt.getTime() / 1000),
    },
  }));
  
  res.json({ data });
}
```

### Trigger Field Options (Dynamic Dropdowns)

```typescript
// POST /ifttt/v1/triggers/power_changed/fields/device_id/options
export async function deviceOptions(req: Request, res: Response) {
  const devices = await prisma.device.findMany({
    where: { userId: req.user.id },
    select: { id: true, friendlyName: true },
  });
  
  res.json({
    data: devices.map(d => ({ label: d.friendlyName, value: d.id })),
  });
}
```

## Action Implementation

### set_power Action

```typescript
// POST /ifttt/v1/actions/set_power
export async function setPowerAction(req: Request, res: Response) {
  const { actionFields } = req.body;
  const { device_id, power_state } = actionFields;
  
  const device = await prisma.device.findFirst({
    where: { id: device_id, userId: req.user.id },
  });
  
  if (!device) {
    return res.status(400).json({ errors: [{ message: 'Device not found' }] });
  }
  
  await prisma.deviceProperty.update({
    where: { deviceId_name: { deviceId: device_id, name: 'power' } },
    data: { value: power_state, lastChangedBy: 'ifttt', lastChangedAt: new Date() },
  });
  
  // Publish for Alexa ChangeReport
  await redis.publish(`device:${device_id}:state`, JSON.stringify({
    property: 'power', value: power_state, source: 'ifttt',
  }));
  
  res.json({
    data: [{ id: `${device_id}-${Date.now()}` }],
  });
}
```

## Realtime API

Push state changes to IFTTT immediately instead of waiting for polling:

```typescript
// services/ifttt-service/src/services/realtime.ts
export async function notifyIfttt(userId: string, triggerId: string) {
  const subscriptions = await prisma.iftttTriggerSubscription.findMany({
    where: { userId, triggerType: triggerId, enabled: true },
  });
  
  for (const sub of subscriptions) {
    await axios.post(
      'https://realtime.ifttt.com/v1/notifications',
      { data: [{ trigger_identity: sub.triggerIdentity }] },
      { headers: { 'IFTTT-Service-Key': process.env.IFTTT_SERVICE_KEY } }
    );
  }
}
```

## Triggers & Actions Summary

### Triggers
| Slug | Description | Fields |
|------|-------------|--------|
| `power_changed` | Device power state changed | `device_id` (optional) |
| `brightness_changed` | Dimmer brightness changed | `device_id` (optional) |
| `sensor_value_changed` | Sensor value changed | `device_id`, `threshold` |

### Actions
| Slug | Description | Fields |
|------|-------------|--------|
| `set_power` | Set device power ON/OFF | `device_id`, `power_state` |
| `set_brightness` | Set dimmer brightness | `device_id`, `brightness` |
| `set_sensor_value` | Set sensor value | `device_id`, `value` |

## Environment Variables

```bash
IFTTT_SERVICE_KEY=your-service-key
IFTTT_CHANNEL_KEY=your-channel-key
