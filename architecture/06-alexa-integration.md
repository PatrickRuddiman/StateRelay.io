# 06 - Alexa Smart Home Skill Integration

## Overview

VirtDevice integrates with Amazon Alexa as a Smart Home Skill, allowing users to:
- Control virtual devices with voice commands
- Include virtual devices in Alexa Routines
- Trigger routines when device states change via proactive ChangeReports

## Alexa Smart Home Skill Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           ALEXA SMART HOME FLOW                              │
└─────────────────────────────────────────────────────────────────────────────┘

User: "Alexa, turn on the living room switch"

┌──────────┐     ┌──────────┐     ┌──────────┐     ┌──────────┐
│   User   │────▶│  Alexa   │────▶│VirtDevice│────▶│ Database │
│  Voice   │     │  Cloud   │     │  Skill   │     │          │
└──────────┘     └──────────┘     └──────────┘     └──────────┘
                      │                                  │
                      │                                  ▼
                      │                           ┌──────────┐
                      │                           │  Redis   │
                      │                           │ (Publish)│
                      │                           └──────────┘
                      │                                  │
                      ▼                                  ▼
               ┌──────────┐                       ┌──────────┐
               │  Alexa   │◀──────────────────────│  IFTTT   │
               │ Response │                       │ Service  │
               └──────────┘                       │ (Notify) │
                                                  └──────────┘
```

## Supported Alexa Interfaces

| Interface | Capability | Voice Commands |
|-----------|------------|----------------|
| `Alexa.PowerController` | ON/OFF control | "Turn on/off [device]" |
| `Alexa.BrightnessController` | 0-100% brightness | "Set [device] to 50%" |
| `Alexa.PercentageController` | Generic 0-100% | "Set [device] to 75 percent" |
| `Alexa.ContactSensor` | Open/Closed state | Trigger only (read-only) |
| `Alexa.MotionSensor` | Motion detected | Trigger only (read-only) |

## Skill Manifest Configuration

```json
{
  "manifest": {
    "publishingInformation": {
      "locales": {
        "en-US": {
          "name": "VirtDevice",
          "summary": "Create virtual devices to bridge Alexa and IFTTT",
          "description": "VirtDevice allows you to create virtual switches, dimmers, and sensors...",
          "examplePhrases": [
            "Alexa, turn on my virtual switch",
            "Alexa, set bedroom dimmer to fifty percent",
            "Alexa, discover my devices"
          ]
        }
      }
    },
    "apis": {
      "smartHome": {
        "endpoint": {
          "uri": "https://alexa.virtdevice.io/api/alexa/smart-home"
        },
        "protocolVersion": "3"
      }
    },
    "permissions": [
      {
        "name": "alexa::async_event:write"
      }
    ],
    "accountLinking": {
      "type": "AUTH_CODE",
      "authorizationUrl": "https://auth.virtdevice.io/oauth/authorize",
      "accessTokenUrl": "https://auth.virtdevice.io/oauth/token",
      "clientId": "alexa-smart-home",
      "scopes": ["read", "write"],
      "accessTokenScheme": "HTTP_BASIC"
    }
  }
}
```

## Directive Handlers

### Discovery Handler

```typescript
// services/alexa-skill/src/handlers/discovery.ts
export async function handleDiscovery(directive: any, userId: string) {
  const devices = await prisma.device.findMany({
    where: { userId },
    include: { properties: true },
  });

  const endpoints = devices.map(device => ({
    endpointId: device.alexaEndpointId,
    manufacturerName: 'VirtDevice',
    friendlyName: device.friendlyName,
    description: `VirtDevice ${device.deviceType}`,
    displayCategories: getDisplayCategories(device.deviceType),
    capabilities: buildCapabilities(device),
    cookie: { deviceId: device.id },
  }));

  return {
    event: {
      header: {
        namespace: 'Alexa.Discovery',
        name: 'Discover.Response',
        payloadVersion: '3',
        messageId: directive.directive.header.messageId,
      },
      payload: { endpoints },
    },
  };
}

function getDisplayCategories(deviceType: string): string[] {
  switch (deviceType) {
    case 'switch': return ['SWITCH'];
    case 'dimmer': return ['LIGHT'];
    case 'sensor': return ['CONTACT_SENSOR'];
    default: return ['OTHER'];
  }
}

function buildCapabilities(device: any) {
  const capabilities = [
    { type: 'AlexaInterface', interface: 'Alexa', version: '3' },
    { type: 'AlexaInterface', interface: 'Alexa.EndpointHealth', version: '3',
      properties: { supported: [{ name: 'connectivity' }], retrievable: true }},
  ];

  if (device.deviceType === 'switch' || device.deviceType === 'dimmer') {
    capabilities.push({
      type: 'AlexaInterface', interface: 'Alexa.PowerController', version: '3',
      properties: { supported: [{ name: 'powerState' }], proactivelyReported: true, retrievable: true },
    });
  }

  if (device.deviceType === 'dimmer') {
    capabilities.push({
      type: 'AlexaInterface', interface: 'Alexa.BrightnessController', version: '3',
      properties: { supported: [{ name: 'brightness' }], proactivelyReported: true, retrievable: true },
    });
  }

  return capabilities;
}
```

### PowerController Handler

```typescript
// services/alexa-skill/src/handlers/power-controller.ts
export async function handlePowerController(directive: any, userId: string) {
  const { endpointId } = directive.directive.endpoint;
  const { name } = directive.directive.header;
  const newValue = name === 'TurnOn' ? 'ON' : 'OFF';

  const device = await prisma.device.findFirst({
    where: { alexaEndpointId: endpointId, userId },
  });

  await prisma.deviceProperty.update({
    where: { deviceId_name: { deviceId: device.id, name: 'power' } },
    data: { value: newValue, lastChangedBy: 'alexa', lastChangedAt: new Date() },
  });

  // Publish state change
  await redis.publish(`device:${device.id}:state`, JSON.stringify({
    property: 'power', value: newValue, source: 'alexa',
  }));

  return {
    event: {
      header: { namespace: 'Alexa', name: 'Response', messageId: directive.directive.header.messageId, payloadVersion: '3' },
      endpoint: { endpointId },
      payload: {},
    },
    context: {
      properties: [
        { namespace: 'Alexa.PowerController', name: 'powerState', value: newValue, timeOfSample: new Date().toISOString(), uncertaintyInMilliseconds: 0 },
      ],
    },
  };
}
```

## Proactive State Reports (ChangeReport)

When device state changes from IFTTT or Web, send ChangeReport to Alexa:

```typescript
// services/alexa-skill/src/services/change-report.ts
export async function sendChangeReport(userId: string, deviceId: string, property: string, value: any) {
  const lwaToken = await getValidLwaToken(userId);
  if (!lwaToken) return;

  const device = await prisma.device.findUnique({ where: { id: deviceId } });

  const payload = {
    event: {
      header: {
        namespace: 'Alexa', name: 'ChangeReport',
        messageId: uuidv4(), payloadVersion: '3',
      },
      endpoint: { scope: { type: 'BearerToken', token: lwaToken }, endpointId: device.alexaEndpointId },
      payload: { change: { cause: { type: 'APP_INTERACTION' },
        properties: [{ namespace: getNamespace(property), name: property, value, timeOfSample: new Date().toISOString(), uncertaintyInMilliseconds: 0 }],
      }},
    },
    context: { properties: [] },
  };

  await axios.post('https://api.amazonalexa.com/v3/events', payload, {
    headers: { Authorization: `Bearer ${lwaToken}` },
  });
}
```

## Environment Variables

```bash
# Alexa Skill
ALEXA_SKILL_ID=amzn1.ask.skill.xxx
LWA_CLIENT_ID=amzn1.application-oa2-client.xxx
LWA_CLIENT_SECRET=your-lwa-secret
