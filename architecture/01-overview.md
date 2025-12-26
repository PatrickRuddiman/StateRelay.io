# 01 - Project Overview

## Executive Summary

**StateRelay** is a multi-tenant SaaS platform that enables virtual smart-home devices whose state can be controlled and monitored across multiple automation ecosystems.

StateRelay provides software-defined switches, dimmers, and sensors that appear as real devices to platforms like Amazon Alexa and IFTTT, allowing state changes in one ecosystem to reliably trigger automations in another.

**Operated by**: Pinnacle Ridge Cloud Solutions LLC

## The Problem

Amazon Alexa and IFTTT are two of the most popular home automation platforms, but they don't natively integrate with each other:

- **Alexa Routines** can be triggered by smart home devices, but IFTTT cannot directly trigger Alexa Routines
- **IFTTT Applets** can control smart devices, but cannot be triggered by Alexa voice commands
- Users who want to use both platforms must find workarounds or purchase physical devices to act as bridges

## The Solution

StateRelay creates **Relay Devices** that appear as real smart home devices to both platforms:

```
┌─────────────────────────────────────────────────────────────────────┐
│                        StateRelay Platform                           │
│                                                                      │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐          │
│  │    Relay     │    │    Relay     │    │    Relay     │          │
│  │   Switch     │    │   Dimmer     │    │   Sensor     │          │
│  │   ON/OFF     │    │   0-100%     │    │   Value      │          │
│  └──────┬───────┘    └──────┬───────┘    └──────┬───────┘          │
│         │                   │                   │                   │
│         └───────────────────┴───────────────────┘                   │
│                             │                                        │
│                    ┌────────┴────────┐                              │
│                    │ StateRelay Core │                              │
│                    │  (PostgreSQL)   │                              │
│                    └────────┬────────┘                              │
│                             │                                        │
│              ┌──────────────┴──────────────┐                        │
│              │                             │                         │
│              ▼                             ▼                         │
│     ┌─────────────────┐          ┌─────────────────┐               │
│     │  Alexa Smart    │          │  IFTTT Service  │               │
│     │  Home Skill     │          │  Integration    │               │
│     └────────┬────────┘          └────────┬────────┘               │
└──────────────┼─────────────────────────────┼────────────────────────┘
               │                             │
               ▼                             ▼
        ┌─────────────┐              ┌─────────────┐
        │    Alexa    │              │    IFTTT    │
        │   Cloud     │              │   Cloud     │
        └─────────────┘              └─────────────┘
```

## Use Cases

### Use Case 1: Alexa Voice → IFTTT Action

> "Alexa, turn on the garage door switch"

1. User speaks to Alexa
2. Alexa sends command to StateRelay
3. StateRelay updates Relay Switch state to ON
4. StateRelay notifies IFTTT via Realtime API (State Relay Event)
5. IFTTT Applet triggers (e.g., opens actual garage door via MyQ)

### Use Case 2: IFTTT Trigger → Alexa Routine

> When Ring doorbell detects motion, turn on porch light

1. Ring detects motion, triggers IFTTT
2. IFTTT Applet sets StateRelay sensor to "motion detected" (State Update)
3. StateRelay sends ChangeReport to Alexa Event Gateway (State Relay Event)
4. Alexa Routine triggers (turn on Philips Hue porch light)

### Use Case 3: Time-based Automation Bridge

> Every day at sunset, enable "Night Mode" across both platforms

1. IFTTT time trigger fires at sunset
2. IFTTT sets StateRelay "Night Mode" switch to ON (State Update)
3. StateRelay notifies Alexa via ChangeReport (State Relay Event)
4. Alexa Routine triggers (dim lights, lock doors, set thermostat)

### Use Case 4: Cross-Platform Device Grouping

> Control devices from both ecosystems with a single command

1. Create StateRelay switch "All Lights"
2. Alexa Routine: When "All Lights" turns ON → turn on all Alexa-connected lights
3. IFTTT Applet: When "All Lights" turns ON → turn on all IFTTT-connected lights
4. Say "Alexa, turn on all lights" → everything turns on

## Core Concepts

### Relay Device

A software-defined device with Relay Properties that can be read and modified. Unlike physical devices:
- No hardware required
- Instant state changes
- Available globally (no network restrictions)
- Unlimited quantity per user

### Relay Property

A named attribute of a Relay Device that holds a value:
- **Power**: ON or OFF
- **Brightness**: 0-100 (integer)
- **Color**: RGB/HSB values
- **Value**: Arbitrary numeric/string value

### Relay Event

An event that fires when a Relay Property changes:
- Alexa: Proactive ChangeReport (State Relay Event) → Alexa Routine
- IFTTT: Realtime API notification (State Relay Event) → IFTTT Applet

### State Update

An operation that modifies a Relay Property:
- Alexa: Voice command → Directive → Property update
- IFTTT: Applet action → API call → Property update

## Business Model

### Subscription Tiers

| Tier | Devices | Price | Features |
|------|---------|-------|----------|
| **Free** | 2 | $0/mo | Basic devices, community support |
| **Pro** | 20 | $4.99/mo | All device types, email support |
| **Business** | Unlimited | $19.99/mo | API access, priority support, custom integrations |

### Revenue Projections

Based on similar services (Stringify, Virtual Buttons for SmartThings):
- Target: 10,000 paying users in Year 1
- Average Revenue Per User (ARPU): $6/month
- Monthly Recurring Revenue (MRR): $60,000

## Success Criteria

1. **Latency**: < 2 seconds from trigger to action completion
2. **Reliability**: 99.9% uptime for core services
3. **Scalability**: Support 100,000+ virtual devices
4. **User Adoption**: 1,000 paying users within 6 months
5. **Platform Approval**: Published Alexa Skill and IFTTT Service

## Project Timeline

| Phase | Duration | Deliverables |
|-------|----------|--------------|
| **Phase 1: Foundation** | 2 weeks | Docker setup, database, auth server |
| **Phase 2: Core API** | 2 weeks | Device CRUD, property management, webhooks |
| **Phase 3: Alexa** | 2 weeks | Smart Home Skill, account linking, proactive events |
| **Phase 4: IFTTT** | 2 weeks | Service creation, triggers, actions, realtime |
| **Phase 5: Frontend** | 2 weeks | Dashboard, device management, billing |
| **Phase 6: Polish** | 2 weeks | Testing, documentation, submission |

**Total Estimated Time**: 12 weeks

## Competitive Landscape

| Product | Pros | Cons |
|---------|------|------|
| **Virtual Buttons (SmartThings)** | Free, easy | SmartThings-only |
| **IFTTT Pro (native)** | Powerful | No Alexa Routine triggers |
| **Home Assistant** | Fully customizable | Complex setup, self-hosted |
| **StateRelay** | Purpose-built state relay bridge | New platform, subscription cost |

## Technical Constraints

### Alexa Requirements
- HTTPS endpoints (TLS 1.2+)
- OAuth 2.0 account linking
- Proactive state reports within 3 seconds
- Self-hosted with Caddy for TLS

### IFTTT Requirements
- HTTPS endpoints (TLS 1.2+)
- OAuth 2.0 authentication
- Response times < 4.5 seconds
- Minimum 12 published Applets

### Infrastructure Requirements
- Domain with valid SSL certificate
- Reliable hosting (99.9% uptime)
- Database with backup/recovery
- Logging and monitoring

## Next Steps

1. ✅ Complete architecture documentation
2. ⏳ Register domain name
3. ⏳ Create Amazon Developer account
4. ⏳ Apply for IFTTT Platform access
5. ⏳ Set up Stripe account
6. ⏳ Begin Phase 1 implementation
