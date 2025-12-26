# 08 - Device Types & Properties

## Virtual Device Types

VirtDevice supports three core device types, each with specific properties and platform mappings.

## Device Type: Switch

A simple on/off toggle device.

### Properties
| Property | Type | Values | Default |
|----------|------|--------|---------|
| `power` | boolean | `ON`, `OFF` | `OFF` |

### Alexa Mapping
- **Display Category**: `SWITCH`
- **Interfaces**: `Alexa.PowerController`
- **Voice Commands**: "Turn on/off [name]"

### IFTTT Mapping
- **Triggers**: `power_changed`
- **Actions**: `set_power`

### JSON Schema
```json
{
  "type": "switch",
  "properties": [
    { "name": "power", "dataType": "boolean", "value": "OFF" }
  ],
  "alexa": {
    "displayCategories": ["SWITCH"],
    "capabilities": ["Alexa.PowerController"]
  }
}
```

---

## Device Type: Dimmer

A dimmable light with power and brightness control.

### Properties
| Property | Type | Values | Default |
|----------|------|--------|---------|
| `power` | boolean | `ON`, `OFF` | `OFF` |
| `brightness` | integer | 0-100 | 0 |

### Alexa Mapping
- **Display Category**: `LIGHT`
- **Interfaces**: `Alexa.PowerController`, `Alexa.BrightnessController`
- **Voice Commands**: "Turn on [name]", "Set [name] to 50%", "Dim [name]"

### IFTTT Mapping
- **Triggers**: `power_changed`, `brightness_changed`
- **Actions**: `set_power`, `set_brightness`

### JSON Schema
```json
{
  "type": "dimmer",
  "properties": [
    { "name": "power", "dataType": "boolean", "value": "OFF" },
    { "name": "brightness", "dataType": "integer", "value": "0", "min": 0, "max": 100 }
  ],
  "alexa": {
    "displayCategories": ["LIGHT"],
    "capabilities": ["Alexa.PowerController", "Alexa.BrightnessController"]
  }
}
```

---

## Device Type: Sensor

A generic value sensor that can trigger routines.

### Properties
| Property | Type | Values | Default |
|----------|------|--------|---------|
| `value` | integer/float | Any number | 0 |
| `state` | string | `open`, `closed`, `detected`, `clear` | `clear` |

### Alexa Mapping
- **Display Category**: `CONTACT_SENSOR` or `MOTION_SENSOR`
- **Interfaces**: `Alexa.ContactSensor` or `Alexa.MotionSensor`
- **Voice Commands**: Read-only, triggers routines

### IFTTT Mapping
- **Triggers**: `sensor_value_changed`, `sensor_state_changed`
- **Actions**: `set_sensor_value`, `set_sensor_state`

### JSON Schema
```json
{
  "type": "sensor",
  "properties": [
    { "name": "value", "dataType": "integer", "value": "0" },
    { "name": "state", "dataType": "string", "value": "clear", "allowedValues": ["open", "closed", "detected", "clear"] }
  ],
  "alexa": {
    "displayCategories": ["CONTACT_SENSOR"],
    "capabilities": ["Alexa.ContactSensor"]
  }
}
```

---

## Device Templates

When users create a device, auto-populate properties based on type:

```typescript
const DEVICE_TEMPLATES = {
  switch: {
    properties: [
      { name: 'power', dataType: 'boolean', value: 'OFF', alexaInterface: 'Alexa.PowerController', alexaPropertyName: 'powerState' },
    ],
    alexaDisplayCategories: ['SWITCH'],
  },
  dimmer: {
    properties: [
      { name: 'power', dataType: 'boolean', value: 'OFF', alexaInterface: 'Alexa.PowerController', alexaPropertyName: 'powerState' },
      { name: 'brightness', dataType: 'integer', value: '0', minValue: 0, maxValue: 100, alexaInterface: 'Alexa.BrightnessController', alexaPropertyName: 'brightness' },
    ],
    alexaDisplayCategories: ['LIGHT'],
  },
  sensor: {
    properties: [
      { name: 'value', dataType: 'integer', value: '0' },
      { name: 'state', dataType: 'string', value: 'clear', allowedValues: ['open', 'closed', 'detected', 'clear'], alexaInterface: 'Alexa.ContactSensor', alexaPropertyName: 'detectionState' },
    ],
    alexaDisplayCategories: ['CONTACT_SENSOR'],
  },
};
```

---

## Property Value Mapping

### Alexa Value Mapping

| VirtDevice Value | Alexa Value |
|------------------|-------------|
| `ON` | `ON` |
| `OFF` | `OFF` |
| `0-100` (brightness) | `0-100` |
| `open` | `DETECTED` |
| `closed` | `NOT_DETECTED` |
| `detected` | `DETECTED` |
| `clear` | `NOT_DETECTED` |

### IFTTT Value Mapping

| VirtDevice Value | IFTTT Value |
|------------------|-------------|
| `ON` | `"on"` |
| `OFF` | `"off"` |
| `0-100` | `0-100` (number) |
| `open`/`closed` | `"open"`, `"closed"` |
