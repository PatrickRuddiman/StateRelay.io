# 04 - Database Design

## Overview

VirtDevice uses PostgreSQL 16 as the primary data store with Row-Level Security (RLS) for multi-tenant isolation. The schema is designed to support:

- Multi-tenant user management
- Virtual device and property storage
- OAuth 2.0 token management for Alexa and IFTTT
- Subscription and billing tracking
- Activity logging and auditing

## Entity Relationship Diagram

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                                    ENTITY RELATIONSHIPS                                  │
└─────────────────────────────────────────────────────────────────────────────────────────┘

┌──────────────┐       ┌──────────────┐       ┌──────────────┐
│   tenants    │       │    users     │       │   devices    │
├──────────────┤       ├──────────────┤       ├──────────────┤
│ id (PK)      │──┐    │ id (PK)      │──┐    │ id (PK)      │
│ name         │  │    │ tenant_id(FK)│◄─┘    │ tenant_id(FK)│
│ slug         │  │    │ email        │  │    │ user_id (FK) │◄──┐
│ created_at   │  └───▶│ password_hash│  │    │ name         │   │
│ updated_at   │       │ role         │  └───▶│ device_type  │   │
└──────────────┘       │ created_at   │       │ friendly_name│   │
                       └──────────────┘       │ created_at   │   │
                              │               └──────────────┘   │
                              │                      │           │
                              ▼                      ▼           │
                       ┌──────────────┐       ┌──────────────┐   │
                       │subscriptions │       │  device_     │   │
                       ├──────────────┤       │  properties  │   │
                       │ id (PK)      │       ├──────────────┤   │
                       │ user_id (FK) │       │ id (PK)      │   │
                       │ stripe_sub_id│       │ device_id(FK)│   │
                       │ tier         │       │ name         │   │
                       │ status       │       │ value        │   │
                       │ current_period│       │ data_type    │   │
                       └──────────────┘       │ updated_at   │   │
                                              └──────────────┘   │
                                                                 │
┌──────────────┐       ┌──────────────┐       ┌──────────────┐   │
│oauth_clients │       │ oauth_tokens │       │alexa_lwa_    │   │
├──────────────┤       ├──────────────┤       │   tokens     │   │
│ id (PK)      │──┐    │ id (PK)      │       ├──────────────┤   │
│ client_id    │  │    │ user_id (FK) │◄──────│ id (PK)      │   │
│ client_secret│  │    │ client_id(FK)│◄──┘   │ user_id (FK) │───┘
│ name         │  │    │ access_token │       │ access_token │
│ type         │  │    │ refresh_token│       │ refresh_token│
│ redirect_uris│  └───▶│ expires_at   │       │ expires_at   │
└──────────────┘       │ scope        │       │ region       │
                       └──────────────┘       └──────────────┘

┌──────────────┐       ┌──────────────┐       ┌──────────────┐
│ifttt_trigger │       │   webhooks   │       │activity_logs │
│_subscriptions│       ├──────────────┤       ├──────────────┤
├──────────────┤       │ id (PK)      │       │ id (PK)      │
│ id (PK)      │       │ device_id(FK)│       │ tenant_id(FK)│
│ user_id (FK) │       │ url          │       │ user_id (FK) │
│ trigger_type │       │ events[]     │       │ device_id(FK)│
│ trigger_fields│       │ secret       │       │ action       │
│ trigger_ident│       │ enabled      │       │ details      │
│ created_at   │       │ created_at   │       │ ip_address   │
└──────────────┘       └──────────────┘       │ created_at   │
                                              └──────────────┘
```

## Complete Schema Definition

### Core Tables

```sql
-- ============================================
-- EXTENSIONS
-- ============================================
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pgcrypto";

-- ============================================
-- ENUMS
-- ============================================
CREATE TYPE user_role AS ENUM ('owner', 'admin', 'member');
CREATE TYPE device_type AS ENUM ('switch', 'dimmer', 'sensor');
CREATE TYPE property_data_type AS ENUM ('boolean', 'integer', 'float', 'string');
CREATE TYPE subscription_tier AS ENUM ('free', 'pro', 'business');
CREATE TYPE subscription_status AS ENUM ('active', 'past_due', 'canceled', 'trialing');
CREATE TYPE oauth_client_type AS ENUM ('web', 'alexa', 'ifttt', 'mobile');
CREATE TYPE activity_action AS ENUM (
    'device_created', 'device_updated', 'device_deleted',
    'property_changed', 'user_login', 'user_logout',
    'subscription_created', 'subscription_updated',
    'alexa_command', 'ifttt_trigger', 'ifttt_action'
);

-- ============================================
-- TENANTS TABLE
-- ============================================
CREATE TABLE tenants (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name VARCHAR(255) NOT NULL,
    slug VARCHAR(100) UNIQUE NOT NULL,
    settings JSONB DEFAULT '{}',
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- Index for slug lookups
CREATE INDEX idx_tenants_slug ON tenants(slug);

-- ============================================
-- USERS TABLE
-- ============================================
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    email VARCHAR(255) NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    name VARCHAR(255),
    role user_role DEFAULT 'member',
    email_verified BOOLEAN DEFAULT FALSE,
    email_verification_token VARCHAR(255),
    password_reset_token VARCHAR(255),
    password_reset_expires TIMESTAMP WITH TIME ZONE,
    last_login_at TIMESTAMP WITH TIME ZONE,
    settings JSONB DEFAULT '{}',
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    
    CONSTRAINT unique_email_per_tenant UNIQUE (tenant_id, email)
);

-- Indexes
CREATE INDEX idx_users_tenant_id ON users(tenant_id);
CREATE INDEX idx_users_email ON users(email);

-- ============================================
-- DEVICES TABLE
-- ============================================
CREATE TABLE devices (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    name VARCHAR(100) NOT NULL,
    friendly_name VARCHAR(255) NOT NULL,
    device_type device_type NOT NULL,
    description TEXT,
    manufacturer VARCHAR(100) DEFAULT 'VirtDevice',
    model VARCHAR(100),
    
    -- Alexa-specific fields
    alexa_endpoint_id VARCHAR(255) UNIQUE,
    alexa_display_categories TEXT[] DEFAULT '{}',
    alexa_capabilities JSONB DEFAULT '[]',
    
    -- IFTTT-specific fields
    ifttt_device_id VARCHAR(255) UNIQUE,
    
    -- Metadata
    is_online BOOLEAN DEFAULT TRUE,
    last_state_change TIMESTAMP WITH TIME ZONE,
    metadata JSONB DEFAULT '{}',
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    
    CONSTRAINT unique_device_name_per_user UNIQUE (user_id, name)
);

-- Indexes
CREATE INDEX idx_devices_tenant_id ON devices(tenant_id);
CREATE INDEX idx_devices_user_id ON devices(user_id);
CREATE INDEX idx_devices_type ON devices(device_type);
CREATE INDEX idx_devices_alexa_endpoint ON devices(alexa_endpoint_id);

-- ============================================
-- DEVICE PROPERTIES TABLE
-- ============================================
CREATE TABLE device_properties (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    device_id UUID NOT NULL REFERENCES devices(id) ON DELETE CASCADE,
    name VARCHAR(100) NOT NULL,
    value TEXT,
    data_type property_data_type NOT NULL,
    
    -- Constraints for the property
    min_value NUMERIC,
    max_value NUMERIC,
    allowed_values TEXT[],
    unit VARCHAR(50),
    
    -- Alexa mapping
    alexa_interface VARCHAR(100),
    alexa_property_name VARCHAR(100),
    
    -- Tracking
    last_changed_by VARCHAR(50), -- 'alexa', 'ifttt', 'web', 'api'
    last_changed_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    
    CONSTRAINT unique_property_per_device UNIQUE (device_id, name)
);

-- Indexes
CREATE INDEX idx_properties_device_id ON device_properties(device_id);
CREATE INDEX idx_properties_name ON device_properties(name);

-- ============================================
-- OAUTH CLIENTS TABLE
-- ============================================
CREATE TABLE oauth_clients (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    client_id VARCHAR(255) UNIQUE NOT NULL,
    client_secret VARCHAR(255) NOT NULL,
    name VARCHAR(255) NOT NULL,
    type oauth_client_type NOT NULL,
    redirect_uris TEXT[] NOT NULL,
    allowed_scopes TEXT[] DEFAULT ARRAY['read', 'write'],
    access_token_lifetime INTEGER DEFAULT 3600, -- seconds
    refresh_token_lifetime INTEGER DEFAULT 0, -- 0 = non-expiring
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- Index
CREATE INDEX idx_oauth_clients_client_id ON oauth_clients(client_id);

-- ============================================
-- OAUTH TOKENS TABLE
-- ============================================
CREATE TABLE oauth_tokens (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    client_id UUID NOT NULL REFERENCES oauth_clients(id) ON DELETE CASCADE,
    access_token VARCHAR(512) UNIQUE NOT NULL,
    access_token_expires_at TIMESTAMP WITH TIME ZONE NOT NULL,
    refresh_token VARCHAR(512) UNIQUE,
    refresh_token_expires_at TIMESTAMP WITH TIME ZONE,
    scope TEXT[] DEFAULT ARRAY['read', 'write'],
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- Indexes
CREATE INDEX idx_oauth_tokens_user_id ON oauth_tokens(user_id);
CREATE INDEX idx_oauth_tokens_access_token ON oauth_tokens(access_token);
CREATE INDEX idx_oauth_tokens_refresh_token ON oauth_tokens(refresh_token);

-- ============================================
-- ALEXA LWA TOKENS TABLE
-- (Login with Amazon tokens for Event Gateway)
-- ============================================
CREATE TABLE alexa_lwa_tokens (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    access_token TEXT NOT NULL,
    refresh_token TEXT NOT NULL,
    access_token_expires_at TIMESTAMP WITH TIME ZONE NOT NULL,
    token_type VARCHAR(50) DEFAULT 'Bearer',
    scope TEXT,
    region VARCHAR(20) DEFAULT 'NA', -- NA, EU, FE
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    
    CONSTRAINT unique_lwa_per_user_region UNIQUE (user_id, region)
);

-- Index
CREATE INDEX idx_alexa_lwa_user_id ON alexa_lwa_tokens(user_id);

-- ============================================
-- IFTTT TRIGGER SUBSCRIPTIONS TABLE
-- ============================================
CREATE TABLE ifttt_trigger_subscriptions (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    trigger_type VARCHAR(100) NOT NULL,
    trigger_fields JSONB NOT NULL DEFAULT '{}',
    trigger_identity VARCHAR(255) UNIQUE NOT NULL,
    ifttt_source JSONB, -- Stores IFTTT source info for skip_check
    enabled BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- Indexes
CREATE INDEX idx_ifttt_subs_user_id ON ifttt_trigger_subscriptions(user_id);
CREATE INDEX idx_ifttt_subs_trigger_type ON ifttt_trigger_subscriptions(trigger_type);
CREATE INDEX idx_ifttt_subs_trigger_identity ON ifttt_trigger_subscriptions(trigger_identity);

-- ============================================
-- SUBSCRIPTIONS TABLE (Stripe)
-- ============================================
CREATE TABLE subscriptions (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    stripe_customer_id VARCHAR(255),
    stripe_subscription_id VARCHAR(255) UNIQUE,
    tier subscription_tier DEFAULT 'free',
    status subscription_status DEFAULT 'active',
    
    -- Plan details
    max_devices INTEGER DEFAULT 2,
    features JSONB DEFAULT '{}',
    
    -- Billing period
    current_period_start TIMESTAMP WITH TIME ZONE,
    current_period_end TIMESTAMP WITH TIME ZONE,
    cancel_at_period_end BOOLEAN DEFAULT FALSE,
    canceled_at TIMESTAMP WITH TIME ZONE,
    
    -- Trial
    trial_start TIMESTAMP WITH TIME ZONE,
    trial_end TIMESTAMP WITH TIME ZONE,
    
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- Indexes
CREATE INDEX idx_subscriptions_user_id ON subscriptions(user_id);
CREATE INDEX idx_subscriptions_stripe_customer ON subscriptions(stripe_customer_id);
CREATE INDEX idx_subscriptions_stripe_sub ON subscriptions(stripe_subscription_id);

-- ============================================
-- WEBHOOKS TABLE
-- ============================================
CREATE TABLE webhooks (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    device_id UUID REFERENCES devices(id) ON DELETE CASCADE, -- NULL = all devices
    url VARCHAR(2048) NOT NULL,
    events TEXT[] NOT NULL, -- ['property_changed', 'device_online', etc.]
    secret VARCHAR(255) NOT NULL, -- For HMAC signature
    headers JSONB DEFAULT '{}', -- Custom headers to send
    enabled BOOLEAN DEFAULT TRUE,
    last_triggered_at TIMESTAMP WITH TIME ZONE,
    last_status_code INTEGER,
    failure_count INTEGER DEFAULT 0,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- Indexes
CREATE INDEX idx_webhooks_user_id ON webhooks(user_id);
CREATE INDEX idx_webhooks_device_id ON webhooks(device_id);

-- ============================================
-- ACTIVITY LOGS TABLE
-- ============================================
CREATE TABLE activity_logs (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    user_id UUID REFERENCES users(id) ON DELETE SET NULL,
    device_id UUID REFERENCES devices(id) ON DELETE SET NULL,
    action activity_action NOT NULL,
    details JSONB DEFAULT '{}',
    source VARCHAR(50), -- 'web', 'alexa', 'ifttt', 'api'
    ip_address INET,
    user_agent TEXT,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- Indexes
CREATE INDEX idx_activity_tenant_id ON activity_logs(tenant_id);
CREATE INDEX idx_activity_user_id ON activity_logs(user_id);
CREATE INDEX idx_activity_device_id ON activity_logs(device_id);
CREATE INDEX idx_activity_created_at ON activity_logs(created_at DESC);

-- Partition by month for large-scale deployments
-- CREATE TABLE activity_logs_2025_01 PARTITION OF activity_logs
--     FOR VALUES FROM ('2025-01-01') TO ('2025-02-01');
```

### Row-Level Security (RLS)

```sql
-- ============================================
-- ROW-LEVEL SECURITY POLICIES
-- ============================================

-- Enable RLS on all multi-tenant tables
ALTER TABLE users ENABLE ROW LEVEL SECURITY;
ALTER TABLE devices ENABLE ROW LEVEL SECURITY;
ALTER TABLE device_properties ENABLE ROW LEVEL SECURITY;
ALTER TABLE oauth_tokens ENABLE ROW LEVEL SECURITY;
ALTER TABLE alexa_lwa_tokens ENABLE ROW LEVEL SECURITY;
ALTER TABLE ifttt_trigger_subscriptions ENABLE ROW LEVEL SECURITY;
ALTER TABLE subscriptions ENABLE ROW LEVEL SECURITY;
ALTER TABLE webhooks ENABLE ROW LEVEL SECURITY;
ALTER TABLE activity_logs ENABLE ROW LEVEL SECURITY;

-- Create application role
CREATE ROLE virtdevice_app;

-- Users policy: Users can only see users in their tenant
CREATE POLICY users_tenant_isolation ON users
    FOR ALL
    TO virtdevice_app
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);

-- Devices policy: Users can only see their own devices
CREATE POLICY devices_user_isolation ON devices
    FOR ALL
    TO virtdevice_app
    USING (user_id = current_setting('app.current_user_id')::UUID);

-- Device properties: Access through device ownership
CREATE POLICY properties_device_isolation ON device_properties
    FOR ALL
    TO virtdevice_app
    USING (
        device_id IN (
            SELECT id FROM devices 
            WHERE user_id = current_setting('app.current_user_id')::UUID
        )
    );

-- OAuth tokens: Users can only see their own tokens
CREATE POLICY tokens_user_isolation ON oauth_tokens
    FOR ALL
    TO virtdevice_app
    USING (user_id = current_setting('app.current_user_id')::UUID);

-- Alexa LWA tokens: Users can only see their own tokens
CREATE POLICY lwa_tokens_user_isolation ON alexa_lwa_tokens
    FOR ALL
    TO virtdevice_app
    USING (user_id = current_setting('app.current_user_id')::UUID);

-- IFTTT subscriptions: Users can only see their own subscriptions
CREATE POLICY ifttt_subs_user_isolation ON ifttt_trigger_subscriptions
    FOR ALL
    TO virtdevice_app
    USING (user_id = current_setting('app.current_user_id')::UUID);

-- Subscriptions: Users can only see their own subscription
CREATE POLICY subscriptions_user_isolation ON subscriptions
    FOR ALL
    TO virtdevice_app
    USING (user_id = current_setting('app.current_user_id')::UUID);

-- Webhooks: Users can only see their own webhooks
CREATE POLICY webhooks_user_isolation ON webhooks
    FOR ALL
    TO virtdevice_app
    USING (user_id = current_setting('app.current_user_id')::UUID);

-- Activity logs: Users can only see logs in their tenant
CREATE POLICY activity_tenant_isolation ON activity_logs
    FOR ALL
    TO virtdevice_app
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);
```

### Functions and Triggers

```sql
-- ============================================
-- HELPER FUNCTIONS
-- ============================================

-- Update updated_at timestamp automatically
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = CURRENT_TIMESTAMP;
    RETURN NEW;
END;
$$ language 'plpgsql';

-- Apply to all tables with updated_at
CREATE TRIGGER update_tenants_updated_at
    BEFORE UPDATE ON tenants
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_users_updated_at
    BEFORE UPDATE ON users
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_devices_updated_at
    BEFORE UPDATE ON devices
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_subscriptions_updated_at
    BEFORE UPDATE ON subscriptions
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_webhooks_updated_at
    BEFORE UPDATE ON webhooks
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

-- ============================================
-- DEVICE STATE CHANGE NOTIFICATION
-- ============================================

-- Notify on property changes (for Redis pub/sub bridge)
CREATE OR REPLACE FUNCTION notify_property_change()
RETURNS TRIGGER AS $$
DECLARE
    device_record RECORD;
    payload JSON;
BEGIN
    -- Get device info
    SELECT id, tenant_id, user_id, name, device_type 
    INTO device_record 
    FROM devices 
    WHERE id = NEW.device_id;
    
    -- Build payload
    payload := json_build_object(
        'device_id', NEW.device_id,
        'tenant_id', device_record.tenant_id,
        'user_id', device_record.user_id,
        'property_name', NEW.name,
        'old_value', OLD.value,
        'new_value', NEW.value,
        'data_type', NEW.data_type,
        'changed_by', NEW.last_changed_by,
        'changed_at', NEW.last_changed_at
    );
    
    -- Send notification
    PERFORM pg_notify('property_changes', payload::text);
    
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER property_change_notification
    AFTER UPDATE ON device_properties
    FOR EACH ROW
    WHEN (OLD.value IS DISTINCT FROM NEW.value)
    EXECUTE FUNCTION notify_property_change();

-- ============================================
-- DEVICE COUNT VALIDATION
-- ============================================

CREATE OR REPLACE FUNCTION check_device_limit()
RETURNS TRIGGER AS $$
DECLARE
    current_count INTEGER;
    max_allowed INTEGER;
BEGIN
    -- Get current device count for user
    SELECT COUNT(*) INTO current_count
    FROM devices
    WHERE user_id = NEW.user_id;
    
    -- Get max allowed from subscription
    SELECT COALESCE(max_devices, 2) INTO max_allowed
    FROM subscriptions
    WHERE user_id = NEW.user_id AND status = 'active';
    
    -- If no subscription, use free tier limit
    IF max_allowed IS NULL THEN
        max_allowed := 2;
    END IF;
    
    -- Check limit
    IF current_count >= max_allowed THEN
        RAISE EXCEPTION 'Device limit reached. You can create up to % devices with your current plan.', max_allowed;
    END IF;
    
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER enforce_device_limit
    BEFORE INSERT ON devices
    FOR EACH ROW
    EXECUTE FUNCTION check_device_limit();

-- ============================================
-- GENERATE UNIQUE IDS
-- ============================================

-- Generate Alexa endpoint ID
CREATE OR REPLACE FUNCTION generate_alexa_endpoint_id()
RETURNS TRIGGER AS $$
BEGIN
    IF NEW.alexa_endpoint_id IS NULL THEN
        NEW.alexa_endpoint_id := 'vd-' || NEW.id::text;
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER set_alexa_endpoint_id
    BEFORE INSERT ON devices
    FOR EACH ROW
    EXECUTE FUNCTION generate_alexa_endpoint_id();

-- Generate IFTTT device ID
CREATE OR REPLACE FUNCTION generate_ifttt_device_id()
RETURNS TRIGGER AS $$
BEGIN
    IF NEW.ifttt_device_id IS NULL THEN
        NEW.ifttt_device_id := 'vd_' || replace(NEW.id::text, '-', '');
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER set_ifttt_device_id
    BEFORE INSERT ON devices
    FOR EACH ROW
    EXECUTE FUNCTION generate_ifttt_device_id();
```

### Seed Data

```sql
-- ============================================
-- SEED DATA
-- ============================================

-- Insert OAuth clients
INSERT INTO oauth_clients (client_id, client_secret, name, type, redirect_uris, allowed_scopes)
VALUES
    (
        'virtdevice-web',
        '$2b$10$xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx', -- bcrypt hash
        'VirtDevice Web Dashboard',
        'web',
        ARRAY['https://app.virtdevice.io/auth/callback'],
        ARRAY['read', 'write', 'admin']
    ),
    (
        'alexa-smart-home',
        '$2b$10$xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx', -- bcrypt hash
        'Amazon Alexa',
        'alexa',
        ARRAY['https://pitangui.amazon.com/api/skill/link/xxx', 'https://layla.amazon.com/api/skill/link/xxx'],
        ARRAY['read', 'write']
    ),
    (
        'ifttt-service',
        '$2b$10$xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx', -- bcrypt hash
        'IFTTT Service',
        'ifttt',
        ARRAY['https://ifttt.com/channels/xxx/authorize'],
        ARRAY['read', 'write']
    );

-- Insert default device templates (for reference)
-- This could be stored in code instead
```

## Prisma Schema

For the Node.js application, use Prisma ORM:

```prisma
// prisma/schema.prisma

generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

enum UserRole {
  owner
  admin
  member
}

enum DeviceType {
  switch
  dimmer
  sensor
}

enum PropertyDataType {
  boolean
  integer
  float
  string
}

enum SubscriptionTier {
  free
  pro
  business
}

enum SubscriptionStatus {
  active
  past_due
  canceled
  trialing
}

enum OAuthClientType {
  web
  alexa
  ifttt
  mobile
}

enum ActivityAction {
  device_created
  device_updated
  device_deleted
  property_changed
  user_login
  user_logout
  subscription_created
  subscription_updated
  alexa_command
  ifttt_trigger
  ifttt_action
}

model Tenant {
  id        String   @id @default(uuid())
  name      String
  slug      String   @unique
  settings  Json     @default("{}")
  createdAt DateTime @default(now()) @map("created_at")
  updatedAt DateTime @updatedAt @map("updated_at")

  users        User[]
  devices      Device[]
  activityLogs ActivityLog[]

  @@map("tenants")
}

model User {
  id                     String    @id @default(uuid())
  tenantId               String    @map("tenant_id")
  email                  String
  passwordHash           String    @map("password_hash")
  name                   String?
  role                   UserRole  @default(member)
  emailVerified          Boolean   @default(false) @map("email_verified")
  emailVerificationToken String?   @map("email_verification_token")
  passwordResetToken     String?   @map("password_reset_token")
  passwordResetExpires   DateTime? @map("password_reset_expires")
  lastLoginAt            DateTime? @map("last_login_at")
  settings               Json      @default("{}")
  createdAt              DateTime  @default(now()) @map("created_at")
  updatedAt              DateTime  @updatedAt @map("updated_at")

  tenant                    Tenant                    @relation(fields: [tenantId], references: [id], onDelete: Cascade)
  devices                   Device[]
  oauthTokens               OAuthToken[]
  alexaLwaTokens            AlexaLwaToken[]
  iftttTriggerSubscriptions IftttTriggerSubscription[]
  subscription              Subscription?
  webhooks                  Webhook[]
  activityLogs              ActivityLog[]

  @@unique([tenantId, email])
  @@map("users")
}

model Device {
  id                    String     @id @default(uuid())
  tenantId              String     @map("tenant_id")
  userId                String     @map("user_id")
  name                  String
  friendlyName          String     @map("friendly_name")
  deviceType            DeviceType @map("device_type")
  description           String?
  manufacturer          String     @default("VirtDevice")
  model                 String?
  alexaEndpointId       String?    @unique @map("alexa_endpoint_id")
  alexaDisplayCategories String[]  @map("alexa_display_categories")
  alexaCapabilities     Json       @default("[]") @map("alexa_capabilities")
  iftttDeviceId         String?    @unique @map("ifttt_device_id")
  isOnline              Boolean    @default(true) @map("is_online")
  lastStateChange       DateTime?  @map("last_state_change")
  metadata              Json       @default("{}")
  createdAt             DateTime   @default(now()) @map("created_at")
  updatedAt             DateTime   @updatedAt @map("updated_at")

  tenant       Tenant           @relation(fields: [tenantId], references: [id], onDelete: Cascade)
  user         User             @relation(fields: [userId], references: [id], onDelete: Cascade)
  properties   DeviceProperty[]
  webhooks     Webhook[]
  activityLogs ActivityLog[]

  @@unique([userId, name])
  @@map("devices")
}

model DeviceProperty {
  id                String           @id @default(uuid())
  deviceId          String           @map("device_id")
  name              String
  value             String?
  dataType          PropertyDataType @map("data_type")
  minValue          Float?           @map("min_value")
  maxValue          Float?           @map("max_value")
  allowedValues     String[]         @map("allowed_values")
  unit              String?
  alexaInterface    String?          @map("alexa_interface")
  alexaPropertyName String?          @map("alexa_property_name")
  lastChangedBy     String?          @map("last_changed_by")
  lastChangedAt     DateTime         @default(now()) @map("last_changed_at")
  createdAt         DateTime         @default(now()) @map("created_at")

  device Device @relation(fields: [deviceId], references: [id], onDelete: Cascade)

  @@unique([deviceId, name])
  @@map("device_properties")
}

model OAuthClient {
  id                   String          @id @default(uuid())
  clientId             String          @unique @map("client_id")
  clientSecret         String          @map("client_secret")
  name                 String
  type                 OAuthClientType
  redirectUris         String[]        @map("redirect_uris")
  allowedScopes        String[]        @default(["read", "write"]) @map("allowed_scopes")
  accessTokenLifetime  Int             @default(3600) @map("access_token_lifetime")
  refreshTokenLifetime Int             @default(0) @map("refresh_token_lifetime")
  createdAt            DateTime        @default(now()) @map("created_at")
  updatedAt            DateTime        @updatedAt @map("updated_at")

  tokens OAuthToken[]

  @@map("oauth_clients")
}

model OAuthToken {
  id                    String    @id @default(uuid())
  userId                String    @map("user_id")
  clientId              String    @map("client_id")
  accessToken           String    @unique @map("access_token")
  accessTokenExpiresAt  DateTime  @map("access_token_expires_at")
  refreshToken          String?   @unique @map("refresh_token")
  refreshTokenExpiresAt DateTime? @map("refresh_token_expires_at")
  scope                 String[]  @default(["read", "write"])
  createdAt             DateTime  @default(now()) @map("created_at")

  user   User        @relation(fields: [userId], references: [id], onDelete: Cascade)
  client OAuthClient @relation(fields: [clientId], references: [id], onDelete: Cascade)

  @@map("oauth_tokens")
}

model AlexaLwaToken {
  id                   String   @id @default(uuid())
  userId               String   @map("user_id")
  accessToken          String   @map("access_token")
  refreshToken         String   @map("refresh_token")
  accessTokenExpiresAt DateTime @map("access_token_expires_at")
  tokenType            String   @default("Bearer") @map("token_type")
  scope                String?
  region               String   @default("NA")
  createdAt            DateTime @default(now()) @map("created_at")
  updatedAt            DateTime @updatedAt @map("updated_at")

  user User @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@unique([userId, region])
  @@map("alexa_lwa_tokens")
}

model IftttTriggerSubscription {
  id              String   @id @default(uuid())
  userId          String   @map("user_id")
  triggerType     String   @map("trigger_type")
  triggerFields   Json     @default("{}") @map("trigger_fields")
  triggerIdentity String   @unique @map("trigger_identity")
  iftttSource     Json?    @map("ifttt_source")
  enabled         Boolean  @default(true)
  createdAt       DateTime @default(now()) @map("created_at")
  updatedAt       DateTime @updatedAt @map("updated_at")

  user User @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@map("ifttt_trigger_subscriptions")
}

model Subscription {
  id                   String             @id @default(uuid())
  userId               String             @unique @map("user_id")
  stripeCustomerId     String?            @map("stripe_customer_id")
  stripeSubscriptionId String?            @unique @map("stripe_subscription_id")
  tier                 SubscriptionTier   @default(free)
  status               SubscriptionStatus @default(active)
  maxDevices           Int                @default(2) @map("max_devices")
  features             Json               @default("{}")
  currentPeriodStart   DateTime?          @map("current_period_start")
  currentPeriodEnd     DateTime?          @map("current_period_end")
  cancelAtPeriodEnd    Boolean            @default(false) @map("cancel_at_period_end")
  canceledAt           DateTime?          @map("canceled_at")
  trialStart           DateTime?          @map("trial_start")
  trialEnd             DateTime?          @map("trial_end")
  createdAt            DateTime           @default(now()) @map("created_at")
  updatedAt            DateTime           @updatedAt @map("updated_at")

  user User @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@map("subscriptions")
}

model Webhook {
  id              String    @id @default(uuid())
  userId          String    @map("user_id")
  deviceId        String?   @map("device_id")
  url             String
  events          String[]
  secret          String
  headers         Json      @default("{}")
  enabled         Boolean   @default(true)
  lastTriggeredAt DateTime? @map("last_triggered_at")
  lastStatusCode  Int?      @map("last_status_code")
  failureCount    Int       @default(0) @map("failure_count")
  createdAt       DateTime  @default(now()) @map("created_at")
  updatedAt       DateTime  @updatedAt @map("updated_at")

  user   User    @relation(fields: [userId], references: [id], onDelete: Cascade)
  device Device? @relation(fields: [deviceId], references: [id], onDelete: Cascade)

  @@map("webhooks")
}

model ActivityLog {
  id        String         @id @default(uuid())
  tenantId  String         @map("tenant_id")
  userId    String?        @map("user_id")
  deviceId  String?        @map("device_id")
  action    ActivityAction
  details   Json           @default("{}")
  source    String?
  ipAddress String?        @map("ip_address")
  userAgent String?        @map("user_agent")
  createdAt DateTime       @default(now()) @map("created_at")

  tenant Tenant  @relation(fields: [tenantId], references: [id], onDelete: Cascade)
  user   User?   @relation(fields: [userId], references: [id], onDelete: SetNull)
  device Device? @relation(fields: [deviceId], references: [id], onDelete: SetNull)

  @@map("activity_logs")
}
```

## Database Migrations

### Initial Migration

```bash
# Generate migration
npx prisma migrate dev --name init

# Apply to production
npx prisma migrate deploy
```

### Adding RLS Policies

Since Prisma doesn't handle RLS, apply these manually after migration:

```bash
# Apply RLS policies
psql $DATABASE_URL -f prisma/migrations/rls_policies.sql
```

## Query Examples

### Setting RLS Context

```typescript
// Before each request, set the tenant and user context
async function setDbContext(tenantId: string, userId: string) {
  await prisma.$executeRaw`
    SET app.current_tenant_id = ${tenantId};
    SET app.current_user_id = ${userId};
  `;
}
```

### Device Queries

```typescript
// Get all devices for current user (RLS handles isolation)
const devices = await prisma.device.findMany({
  include: {
    properties: true,
  },
});

// Create device with properties
const device = await prisma.device.create({
  data: {
    tenantId: user.tenantId,
    userId: user.id,
    name: 'living-room-switch',
    friendlyName: 'Living Room Switch',
    deviceType: 'switch',
    properties: {
      create: [
        {
          name: 'power',
          value: 'OFF',
          dataType: 'boolean',
          alexaInterface: 'Alexa.PowerController',
          alexaPropertyName: 'powerState',
        },
      ],
    },
  },
  include: {
    properties: true,
  },
});

// Update property value
const property = await prisma.deviceProperty.update({
  where: {
    deviceId_name: {
      deviceId: deviceId,
      name: 'power',
    },
  },
  data: {
    value: 'ON',
    lastChangedBy: 'alexa',
    lastChangedAt: new Date(),
  },
});
```

## Performance Considerations

### Indexes Already Created

- `idx_devices_tenant_id` - Tenant-based queries
- `idx_devices_user_id` - User-based queries
- `idx_devices_alexa_endpoint` - Alexa lookup by endpoint ID
- `idx_properties_device_id` - Property lookups
- `idx_oauth_tokens_access_token` - Token validation
- `idx_activity_created_at` - Time-based log queries

### Additional Recommendations

1. **Connection Pooling**: Use PgBouncer for high-traffic scenarios
2. **Read Replicas**: Offload read queries to replicas
3. **Partitioning**: Partition `activity_logs` by month for large datasets
4. **Vacuum**: Schedule regular VACUUM ANALYZE for optimal performance
