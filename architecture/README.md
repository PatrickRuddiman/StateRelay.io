# StateRelay Architecture Documentation

This folder contains comprehensive architecture documentation for StateRelay - a multi-tenant SaaS platform that enables virtual smart-home devices whose state can be controlled and monitored across multiple automation ecosystems.

**Operated by**: Pinnacle Ridge Cloud Solutions LLC

## Document Index

| Document | Description |
|----------|-------------|
| [Overview](./01-overview.md) | High-level system overview and goals |
| [Domain & Naming](./02-domain-naming.md) | Domain name suggestions and naming conventions |
| [System Architecture](./03-system-architecture.md) | Detailed system architecture with diagrams |
| [Database Design](./04-database-design.md) | PostgreSQL schema and multi-tenancy |
| [Authentication](./05-authentication.md) | OAuth 2.0 implementation for Alexa & IFTTT |
| [Alexa Integration](./06-alexa-integration.md) | Smart Home Skill implementation details |
| [IFTTT Integration](./07-ifttt-integration.md) | IFTTT Service implementation details |
| [Device Types](./08-device-types.md) | Virtual device specifications |
| [Event System](./09-event-system.md) | Real-time event flow and pub/sub |
| [API Reference](./10-api-reference.md) | REST API endpoints |
| [Docker & Deployment](./11-docker-deployment.md) | Container architecture and CI/CD |
| [Billing & Stripe](./12-billing-stripe.md) | Payment integration |
| [Security](./13-security.md) | Security considerations and practices |
| [Brand Guidelines](./14-brand-guidelines.md) | Logo, colors, typography, messaging |

## Quick Start

```bash
# Clone the repository
git clone https://github.com/PatrickRuddiman/StateRelay.git
cd StateRelay

# Copy environment template
cp .env.example .env

# Start all services
docker-compose up -d
```

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              CADDY (Reverse Proxy)                          │
│                    TLS Termination • Rate Limiting • Routing                │
└──────────────────────────────────┬──────────────────────────────────────────┘
                                   │
        ┌──────────────────────────┼──────────────────────────────┐
        │                          │                              │
        ▼                          ▼                              ▼
┌───────────────┐        ┌─────────────────┐            ┌─────────────────┐
│   Web App     │        │   API Service   │            │  Alexa Skill    │
│   Dashboard   │        │   REST API      │            │   Endpoint      │
└───────────────┘        └────────┬────────┘            └────────┬────────┘
                                  │                              │
                                  ▼                              │
                         ┌─────────────────┐                     │
                         │  IFTTT Service  │                     │
                         │   Endpoint      │                     │
                         └────────┬────────┘                     │
                                  │                              │
                                  ▼                              ▼
                         ┌─────────────────────────────────────────┐
                         │            Core Services                 │
                         │   Auth • Devices • Events • Billing     │
                         └────────────────────┬────────────────────┘
                                              │
                              ┌───────────────┴───────────────┐
                              ▼                               ▼
                    ┌──────────────────┐            ┌──────────────────┐
                    │   PostgreSQL     │            │     Redis        │
                    │   Primary Data   │            │   Cache/PubSub   │
                    └──────────────────┘            └──────────────────┘
```

## Key Features

- **🔌 Relay Devices**: Create software-defined switches, dimmers, and sensors
- **🗣️ Alexa Integration**: Control devices with voice, trigger Alexa Routines from state changes
- **⚡ IFTTT Integration**: Use device states as triggers, control devices as actions
- **🔄 Bi-directional State Relay**: Changes from either platform instantly reflected in the other
- **👥 Multi-tenant**: Complete user isolation with row-level security
- **💳 Paid Service**: Stripe-powered subscription billing (Pinnacle Ridge Cloud Solutions LLC)
- **🐳 Docker-first**: All services containerized and built via GitHub Actions

## Technology Stack

| Layer | Technology |
|-------|------------|
| Reverse Proxy | Caddy 2 |
| Frontend | Next.js 14 (React) |
| Backend API | Node.js + Express |
| Database | PostgreSQL 16 |
| Cache/PubSub | Redis 7 |
| Authentication | Custom OAuth 2.0 Server |
| Payments | Stripe |
| Containers | Docker + Docker Compose |
| CI/CD | GitHub Actions |
| Container Registry | GitHub Container Registry (ghcr.io) |
| Domain | staterelay.io |
| Company | Pinnacle Ridge Cloud Solutions LLC |
