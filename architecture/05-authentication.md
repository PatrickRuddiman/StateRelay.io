# 05 - Authentication & OAuth 2.0

## Overview

VirtDevice implements a custom OAuth 2.0 authorization server that supports multiple client types:
- **Web Dashboard**: First-party web application login
- **Amazon Alexa**: Account linking for Smart Home Skill
- **IFTTT**: Account linking for IFTTT Service

All three clients use the same OAuth 2.0 server with the Authorization Code Grant flow.

## OAuth 2.0 Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                           OAUTH 2.0 AUTHORIZATION CODE FLOW                              │
└─────────────────────────────────────────────────────────────────────────────────────────┘

┌──────────┐     ┌──────────┐     ┌──────────┐     ┌──────────┐     ┌──────────┐
│  Client  │     │   User   │     │   Auth   │     │   Auth   │     │  Client  │
│  (Alexa/ │     │ Browser  │     │  Server  │     │  Server  │     │ Backend  │
│   IFTTT) │     │          │     │  (Auth)  │     │  (Token) │     │          │
└────┬─────┘     └────┬─────┘     └────┬─────┘     └────┬─────┘     └────┬─────┘
     │                │                │                │                │
     │  1. User enables skill/applet   │                │                │
     ├───────────────▶│                │                │                │
     │                │                │                │                │
     │                │ 2. Redirect to auth endpoint    │                │
     │                │    with client_id, state,       │                │
     │                │    redirect_uri                 │                │
     │                ├───────────────▶│                │                │
     │                │                │                │                │
     │                │                │ 3. Display login page           │
     │                │◀───────────────┤                │                │
     │                │                │                │                │
     │                │ 4. User enters credentials      │                │
     │                ├───────────────▶│                │                │
     │                │                │                │                │
     │                │                │ 5. Validate credentials         │
     │                │                │    Display consent screen       │
     │                │◀───────────────┤                │                │
     │                │                │                │                │
     │                │ 6. User grants permission       │                │
     │                ├───────────────▶│                │                │
     │                │                │                │                │
     │                │ 7. Redirect with authorization code              │
     │                │◀───────────────┤                │                │
     │                │                │                │                │
     │  8. Redirect with code + state  │                │                │
     │◀───────────────┤                │                │                │
     │                │                │                │                │
     │                │                │  9. Exchange code for tokens    │
     ├────────────────┼────────────────┼───────────────▶│                │
     │                │                │                │                │
     │                │                │ 10. Return access + refresh tokens
     │◀───────────────┼────────────────┼────────────────┤                │
     │                │                │                │                │
     │ 11. Use access token for API calls              │                │
     ├────────────────┼────────────────┼────────────────┼───────────────▶│
     │                │                │                │                │
     │                │                │                │ 12. Validate token
     │                │                │                │     Return data
     │◀───────────────┼────────────────┼────────────────┼────────────────┤
     │                │                │                │                │
```

## OAuth 2.0 Endpoints

### Authorization Endpoint

**URL**: `https://auth.virtdevice.io/oauth/authorize`

**Method**: GET

**Parameters**:
| Parameter | Required | Description |
|-----------|----------|-------------|
| `client_id` | Yes | Registered OAuth client ID |
| `response_type` | Yes | Must be `code` |
| `redirect_uri` | Yes | Callback URL (must match registered URIs) |
| `state` | Yes | Anti-CSRF token (returned unchanged) |
| `scope` | No | Space-separated scopes (default: `read write`) |

**Example Request**:
```
GET https://auth.virtdevice.io/oauth/authorize?
  client_id=alexa-smart-home&
  response_type=code&
  redirect_uri=https://pitangui.amazon.com/api/skill/link/xxx&
  state=abc123xyz&
  scope=read%20write
```

**Response**: Redirect to login page, then consent page, then redirect_uri with code

```
HTTP/1.1 302 Found
Location: https://pitangui.amazon.com/api/skill/link/xxx?
  code=AUTH_CODE_HERE&
  state=abc123xyz
```

### Token Endpoint

**URL**: `https://auth.virtdevice.io/oauth/token`

**Method**: POST

**Content-Type**: `application/x-www-form-urlencoded`

#### Grant Type: Authorization Code

**Parameters**:
| Parameter | Required | Description |
|-----------|----------|-------------|
| `grant_type` | Yes | Must be `authorization_code` |
| `code` | Yes | Authorization code from authorize endpoint |
| `redirect_uri` | Yes | Must match the original request |
| `client_id` | Yes | OAuth client ID |
| `client_secret` | Yes | OAuth client secret |

**Example Request**:
```http
POST /oauth/token HTTP/1.1
Host: auth.virtdevice.io
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code&
code=AUTH_CODE_HERE&
redirect_uri=https://pitangui.amazon.com/api/skill/link/xxx&
client_id=alexa-smart-home&
client_secret=SECRET_HERE
```

**Success Response**:
```json
{
  "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "refresh_token": "dGhpcyBpcyBhIHJlZnJlc2ggdG9rZW4...",
  "scope": "read write"
}
```

#### Grant Type: Refresh Token

**Parameters**:
| Parameter | Required | Description |
|-----------|----------|-------------|
| `grant_type` | Yes | Must be `refresh_token` |
| `refresh_token` | Yes | The refresh token |
| `client_id` | Yes | OAuth client ID |
| `client_secret` | Yes | OAuth client secret |

**Example Request**:
```http
POST /oauth/token HTTP/1.1
Host: auth.virtdevice.io
Content-Type: application/x-www-form-urlencoded

grant_type=refresh_token&
refresh_token=dGhpcyBpcyBhIHJlZnJlc2ggdG9rZW4...&
client_id=alexa-smart-home&
client_secret=SECRET_HERE
```

**Success Response**:
```json
{
  "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "refresh_token": "bmV3IHJlZnJlc2ggdG9rZW4...",
  "scope": "read write"
}
```

### User Info Endpoint

**URL**: `https://auth.virtdevice.io/oauth/userinfo`

**Method**: GET

**Headers**:
```
Authorization: Bearer ACCESS_TOKEN_HERE
```

**Response**:
```json
{
  "id": "uuid-here",
  "email": "user@example.com",
  "name": "John Doe"
}
```

### Token Revocation Endpoint

**URL**: `https://auth.virtdevice.io/oauth/revoke`

**Method**: POST

**Parameters**:
| Parameter | Required | Description |
|-----------|----------|-------------|
| `token` | Yes | Access or refresh token to revoke |
| `token_type_hint` | No | `access_token` or `refresh_token` |
| `client_id` | Yes | OAuth client ID |
| `client_secret` | Yes | OAuth client secret |

---

## JWT Token Structure

### Access Token

VirtDevice uses JWT (JSON Web Tokens) for access tokens:

```json
{
  "header": {
    "alg": "HS256",
    "typ": "JWT"
  },
  "payload": {
    "sub": "user-uuid-here",
    "tenant_id": "tenant-uuid-here",
    "client_id": "alexa-smart-home",
    "scope": ["read", "write"],
    "iat": 1704067200,
    "exp": 1704070800,
    "iss": "https://auth.virtdevice.io",
    "aud": "virtdevice-api"
  }
}
```

### Token Payload Fields

| Field | Description |
|-------|-------------|
| `sub` | User ID (UUID) |
| `tenant_id` | Tenant ID for multi-tenant isolation |
| `client_id` | OAuth client that requested the token |
| `scope` | Array of granted scopes |
| `iat` | Issued at timestamp |
| `exp` | Expiration timestamp |
| `iss` | Token issuer (auth server URL) |
| `aud` | Token audience (API server) |

### Refresh Token

Refresh tokens are opaque strings stored in the database:
- Non-expiring (per Alexa/IFTTT requirements)
- Revocable by user or admin
- Tied to specific client and user

---

## Client Configuration

### Alexa Smart Home Skill

**Client ID**: `alexa-smart-home`

**Redirect URIs** (from Amazon Developer Console):
```
https://pitangui.amazon.com/api/skill/link/M3XXXXX
https://layla.amazon.com/api/skill/link/M3XXXXX
https://alexa.amazon.co.jp/api/skill/link/M3XXXXX
```

**Scopes**: `read`, `write`

**Token Lifetime**:
- Access Token: 1 hour (3600 seconds)
- Refresh Token: Non-expiring

### IFTTT Service

**Client ID**: `ifttt-service`

**Redirect URIs** (from IFTTT Platform):
```
https://ifttt.com/channels/virtdevice/authorize
```

**Scopes**: `read`, `write`

**Token Lifetime**:
- Access Token: 1 hour (3600 seconds)
- Refresh Token: Non-expiring

### Web Dashboard

**Client ID**: `virtdevice-web`

**Redirect URIs**:
```
https://app.virtdevice.io/auth/callback
http://localhost:3000/auth/callback  (development only)
```

**Scopes**: `read`, `write`, `admin`

**Token Lifetime**:
- Access Token: 15 minutes (900 seconds)
- Refresh Token: 7 days

---

## Implementation

### Auth Service Structure

```
services/auth/
├── src/
│   ├── index.ts                 # Express app entry point
│   ├── config/
│   │   └── oauth.ts             # OAuth configuration
│   ├── controllers/
│   │   ├── authorize.ts         # Authorization endpoint
│   │   ├── token.ts             # Token endpoint
│   │   ├── userinfo.ts          # User info endpoint
│   │   └── revoke.ts            # Revocation endpoint
│   ├── middleware/
│   │   ├── authenticate.ts      # Verify access tokens
│   │   └── validate-client.ts   # Validate OAuth clients
│   ├── models/
│   │   ├── oauth-client.ts      # OAuth client model
│   │   ├── oauth-token.ts       # Token model
│   │   └── authorization-code.ts # Auth code model
│   ├── services/
│   │   ├── token-service.ts     # Generate/validate tokens
│   │   └── user-service.ts      # User authentication
│   └── views/
│       ├── login.ejs            # Login page
│       └── consent.ejs          # OAuth consent page
├── Dockerfile
└── package.json
```

### Key Code Examples

#### Token Generation

```typescript
// services/auth/src/services/token-service.ts
import jwt from 'jsonwebtoken';
import crypto from 'crypto';
import { prisma } from '../lib/prisma';

interface TokenPayload {
  sub: string;
  tenant_id: string;
  client_id: string;
  scope: string[];
}

export class TokenService {
  private readonly jwtSecret: string;
  private readonly accessTokenLifetime: number;
  
  constructor() {
    this.jwtSecret = process.env.JWT_SECRET!;
    this.accessTokenLifetime = 3600; // 1 hour
  }
  
  async generateAccessToken(payload: TokenPayload): Promise<string> {
    return jwt.sign(
      {
        ...payload,
        aud: 'virtdevice-api',
        iss: 'https://auth.virtdevice.io',
      },
      this.jwtSecret,
      {
        expiresIn: this.accessTokenLifetime,
      }
    );
  }
  
  async generateRefreshToken(): Promise<string> {
    return crypto.randomBytes(64).toString('base64url');
  }
  
  async verifyAccessToken(token: string): Promise<TokenPayload | null> {
    try {
      const payload = jwt.verify(token, this.jwtSecret) as TokenPayload;
      return payload;
    } catch (error) {
      return null;
    }
  }
  
  async createTokenPair(
    userId: string,
    tenantId: string,
    clientId: string,
    scope: string[]
  ): Promise<{ accessToken: string; refreshToken: string; expiresIn: number }> {
    const accessToken = await this.generateAccessToken({
      sub: userId,
      tenant_id: tenantId,
      client_id: clientId,
      scope,
    });
    
    const refreshToken = await this.generateRefreshToken();
    
    // Store tokens in database
    await prisma.oAuthToken.create({
      data: {
        userId,
        clientId,
        accessToken,
        accessTokenExpiresAt: new Date(Date.now() + this.accessTokenLifetime * 1000),
        refreshToken,
        refreshTokenExpiresAt: null, // Non-expiring
        scope,
      },
    });
    
    return {
      accessToken,
      refreshToken,
      expiresIn: this.accessTokenLifetime,
    };
  }
  
  async refreshTokens(
    refreshToken: string,
    clientId: string
  ): Promise<{ accessToken: string; refreshToken: string; expiresIn: number } | null> {
    // Find existing token
    const existingToken = await prisma.oAuthToken.findUnique({
      where: { refreshToken },
      include: { user: true, client: true },
    });
    
    if (!existingToken || existingToken.client.clientId !== clientId) {
      return null;
    }
    
    // Generate new token pair
    const newAccessToken = await this.generateAccessToken({
      sub: existingToken.userId,
      tenant_id: existingToken.user.tenantId,
      client_id: clientId,
      scope: existingToken.scope,
    });
    
    const newRefreshToken = await this.generateRefreshToken();
    
    // Update database
    await prisma.oAuthToken.update({
      where: { id: existingToken.id },
      data: {
        accessToken: newAccessToken,
        accessTokenExpiresAt: new Date(Date.now() + this.accessTokenLifetime * 1000),
        refreshToken: newRefreshToken,
      },
    });
    
    return {
      accessToken: newAccessToken,
      refreshToken: newRefreshToken,
      expiresIn: this.accessTokenLifetime,
    };
  }
}
```

#### Authorization Endpoint

```typescript
// services/auth/src/controllers/authorize.ts
import { Request, Response } from 'express';
import { prisma } from '../lib/prisma';
import crypto from 'crypto';

export async function authorizeHandler(req: Request, res: Response) {
  const { client_id, response_type, redirect_uri, state, scope } = req.query;
  
  // Validate required parameters
  if (!client_id || !response_type || !redirect_uri || !state) {
    return res.status(400).json({ error: 'invalid_request' });
  }
  
  if (response_type !== 'code') {
    return res.status(400).json({ error: 'unsupported_response_type' });
  }
  
  // Validate client
  const client = await prisma.oAuthClient.findUnique({
    where: { clientId: client_id as string },
  });
  
  if (!client) {
    return res.status(400).json({ error: 'invalid_client' });
  }
  
  // Validate redirect URI
  if (!client.redirectUris.includes(redirect_uri as string)) {
    return res.status(400).json({ error: 'invalid_redirect_uri' });
  }
  
  // Store authorization request in session
  req.session.authRequest = {
    clientId: client_id,
    redirectUri: redirect_uri,
    state,
    scope: (scope as string)?.split(' ') || ['read', 'write'],
  };
  
  // Check if user is already logged in
  if (req.session.userId) {
    // Show consent screen
    return res.render('consent', {
      client,
      scope: req.session.authRequest.scope,
    });
  }
  
  // Show login page
  res.render('login', { error: null });
}

export async function authorizeSubmitHandler(req: Request, res: Response) {
  const { email, password } = req.body;
  const authRequest = req.session.authRequest;
  
  if (!authRequest) {
    return res.status(400).json({ error: 'invalid_request' });
  }
  
  // Authenticate user
  const user = await prisma.user.findFirst({
    where: { email },
  });
  
  if (!user || !await verifyPassword(password, user.passwordHash)) {
    return res.render('login', { error: 'Invalid email or password' });
  }
  
  // Store user in session
  req.session.userId = user.id;
  
  // Show consent screen
  const client = await prisma.oAuthClient.findUnique({
    where: { clientId: authRequest.clientId as string },
  });
  
  res.render('consent', {
    client,
    scope: authRequest.scope,
  });
}

export async function consentHandler(req: Request, res: Response) {
  const { approve } = req.body;
  const authRequest = req.session.authRequest;
  const userId = req.session.userId;
  
  if (!authRequest || !userId) {
    return res.status(400).json({ error: 'invalid_request' });
  }
  
  if (approve !== 'true') {
    // User denied access
    const redirectUri = new URL(authRequest.redirectUri as string);
    redirectUri.searchParams.set('error', 'access_denied');
    redirectUri.searchParams.set('state', authRequest.state as string);
    return res.redirect(redirectUri.toString());
  }
  
  // Generate authorization code
  const code = crypto.randomBytes(32).toString('base64url');
  
  // Store authorization code (expires in 10 minutes)
  await prisma.authorizationCode.create({
    data: {
      code,
      userId,
      clientId: authRequest.clientId as string,
      redirectUri: authRequest.redirectUri as string,
      scope: authRequest.scope,
      expiresAt: new Date(Date.now() + 10 * 60 * 1000),
    },
  });
  
  // Clear session
  delete req.session.authRequest;
  
  // Redirect back to client with code
  const redirectUri = new URL(authRequest.redirectUri as string);
  redirectUri.searchParams.set('code', code);
  redirectUri.searchParams.set('state', authRequest.state as string);
  
  res.redirect(redirectUri.toString());
}
```

#### Token Endpoint

```typescript
// services/auth/src/controllers/token.ts
import { Request, Response } from 'express';
import { prisma } from '../lib/prisma';
import { TokenService } from '../services/token-service';

const tokenService = new TokenService();

export async function tokenHandler(req: Request, res: Response) {
  const { grant_type, code, refresh_token, redirect_uri, client_id, client_secret } = req.body;
  
  // Validate client credentials
  const client = await prisma.oAuthClient.findUnique({
    where: { clientId: client_id },
  });
  
  if (!client || client.clientSecret !== client_secret) {
    return res.status(401).json({ error: 'invalid_client' });
  }
  
  if (grant_type === 'authorization_code') {
    return handleAuthorizationCodeGrant(req, res, client);
  } else if (grant_type === 'refresh_token') {
    return handleRefreshTokenGrant(req, res, client);
  } else {
    return res.status(400).json({ error: 'unsupported_grant_type' });
  }
}

async function handleAuthorizationCodeGrant(req: Request, res: Response, client: any) {
  const { code, redirect_uri } = req.body;
  
  // Find and validate authorization code
  const authCode = await prisma.authorizationCode.findUnique({
    where: { code },
    include: { user: true },
  });
  
  if (!authCode) {
    return res.status(400).json({ error: 'invalid_grant' });
  }
  
  if (authCode.clientId !== client.clientId) {
    return res.status(400).json({ error: 'invalid_grant' });
  }
  
  if (authCode.redirectUri !== redirect_uri) {
    return res.status(400).json({ error: 'invalid_grant' });
  }
  
  if (authCode.expiresAt < new Date()) {
    await prisma.authorizationCode.delete({ where: { id: authCode.id } });
    return res.status(400).json({ error: 'invalid_grant' });
  }
  
  // Delete the authorization code (single use)
  await prisma.authorizationCode.delete({ where: { id: authCode.id } });
  
  // Generate tokens
  const tokens = await tokenService.createTokenPair(
    authCode.userId,
    authCode.user.tenantId,
    client.clientId,
    authCode.scope
  );
  
  res.json({
    access_token: tokens.accessToken,
    token_type: 'Bearer',
    expires_in: tokens.expiresIn,
    refresh_token: tokens.refreshToken,
    scope: authCode.scope.join(' '),
  });
}

async function handleRefreshTokenGrant(req: Request, res: Response, client: any) {
  const { refresh_token } = req.body;
  
  const tokens = await tokenService.refreshTokens(refresh_token, client.clientId);
  
  if (!tokens) {
    return res.status(400).json({ error: 'invalid_grant' });
  }
  
  res.json({
    access_token: tokens.accessToken,
    token_type: 'Bearer',
    expires_in: tokens.expiresIn,
    refresh_token: tokens.refreshToken,
  });
}
```

#### Authentication Middleware

```typescript
// services/api/src/middleware/authenticate.ts
import { Request, Response, NextFunction } from 'express';
import jwt from 'jsonwebtoken';

interface JWTPayload {
  sub: string;
  tenant_id: string;
  client_id: string;
  scope: string[];
}

declare global {
  namespace Express {
    interface Request {
      user?: {
        id: string;
        tenantId: string;
        clientId: string;
        scope: string[];
      };
    }
  }
}

export function authenticate(req: Request, res: Response, next: NextFunction) {
  const authHeader = req.headers.authorization;
  
  if (!authHeader || !authHeader.startsWith('Bearer ')) {
    return res.status(401).json({ error: 'missing_token' });
  }
  
  const token = authHeader.substring(7);
  
  try {
    const payload = jwt.verify(token, process.env.JWT_SECRET!) as JWTPayload;
    
    req.user = {
      id: payload.sub,
      tenantId: payload.tenant_id,
      clientId: payload.client_id,
      scope: payload.scope,
    };
    
    next();
  } catch (error) {
    if (error instanceof jwt.TokenExpiredError) {
      return res.status(401).json({ error: 'token_expired' });
    }
    return res.status(401).json({ error: 'invalid_token' });
  }
}

export function requireScope(requiredScope: string) {
  return (req: Request, res: Response, next: NextFunction) => {
    if (!req.user) {
      return res.status(401).json({ error: 'not_authenticated' });
    }
    
    if (!req.user.scope.includes(requiredScope)) {
      return res.status(403).json({ error: 'insufficient_scope' });
    }
    
    next();
  };
}
```

---

## Login with Amazon (LWA) for Alexa Event Gateway

When users link their Alexa account, we also need to obtain tokens for sending proactive events to the Alexa Event Gateway.

### AcceptGrant Flow

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                         ALEXA ACCEPT GRANT FLOW                               │
└──────────────────────────────────────────────────────────────────────────────┘

┌──────────┐     ┌──────────┐     ┌──────────┐     ┌──────────┐
│  Alexa   │     │  Alexa   │     │   LWA    │     │ VirtDev  │
│  Cloud   │     │  Skill   │     │  OAuth   │     │ Database │
└────┬─────┘     └────┬─────┘     └────┬─────┘     └────┬─────┘
     │                │                │                │
     │  1. AcceptGrant directive      │                │
     │    (contains LWA auth code)    │                │
     ├───────────────▶│                │                │
     │                │                │                │
     │                │ 2. Exchange auth code for tokens
     │                ├───────────────▶│                │
     │                │                │                │
     │                │ 3. Return access + refresh tokens
     │                │◀───────────────┤                │
     │                │                │                │
     │                │ 4. Store tokens │                │
     │                ├────────────────┼───────────────▶│
     │                │                │                │
     │  5. AcceptGrant.Response       │                │
     │◀───────────────┤                │                │
     │                │                │                │
```

### AcceptGrant Handler

```typescript
// services/alexa-skill/src/handlers/accept-grant.ts
import axios from 'axios';
import { prisma } from '../lib/prisma';

interface AcceptGrantDirective {
  directive: {
    header: {
      namespace: 'Alexa.Authorization';
      name: 'AcceptGrant';
      messageId: string;
    };
    payload: {
      grant: {
        type: 'OAuth2.AuthorizationCode';
        code: string;
      };
      grantee: {
        type: 'BearerToken';
        token: string; // Our access token
      };
    };
  };
}

export async function handleAcceptGrant(directive: AcceptGrantDirective) {
  const { grant, grantee } = directive.directive.payload;
  
  // Validate our access token to get user info
  const user = await getUserFromToken(grantee.token);
  if (!user) {
    return {
      event: {
        header: {
          namespace: 'Alexa.Authorization',
          name: 'ErrorResponse',
          messageId: directive.directive.header.messageId,
          payloadVersion: '3',
        },
        payload: {
          type: 'ACCEPT_GRANT_FAILED',
          message: 'Invalid user token',
        },
      },
    };
  }
  
  // Exchange LWA authorization code for tokens
  const lwaTokens = await exchangeLwaCode(grant.code);
  
  if (!lwaTokens) {
    return {
      event: {
        header: {
          namespace: 'Alexa.Authorization',
          name: 'ErrorResponse',
          messageId: directive.directive.header.messageId,
          payloadVersion: '3',
        },
        payload: {
          type: 'ACCEPT_GRANT_FAILED',
          message: 'Failed to exchange LWA code',
        },
      },
    };
  }
  
  // Store LWA tokens
  await prisma.alexaLwaToken.upsert({
    where: {
      userId_region: {
        userId: user.id,
        region: 'NA', // Determine from directive if multi-region
      },
    },
    create: {
      userId: user.id,
      accessToken: lwaTokens.access_token,
      refreshToken: lwaTokens.refresh_token,
      accessTokenExpiresAt: new Date(Date.now() + lwaTokens.expires_in * 1000),
      tokenType: lwaTokens.token_type,
      scope: lwaTokens.scope,
      region: 'NA',
    },
    update: {
      accessToken: lwaTokens.access_token,
      refreshToken: lwaTokens.refresh_token,
      accessTokenExpiresAt: new Date(Date.now() + lwaTokens.expires_in * 1000),
    },
  });
  
  return {
    event: {
      header: {
        namespace: 'Alexa.Authorization',
        name: 'AcceptGrant.Response',
        messageId: directive.directive.header.messageId,
        payloadVersion: '3',
      },
      payload: {},
    },
  };
}

async function exchangeLwaCode(code: string) {
  try {
    const response = await axios.post(
      'https://api.amazon.com/auth/o2/token',
      new URLSearchParams({
        grant_type: 'authorization_code',
        code,
        client_id: process.env.LWA_CLIENT_ID!,
        client_secret: process.env.LWA_CLIENT_SECRET!,
      }),
      {
        headers: {
          'Content-Type': 'application/x-www-form-urlencoded',
        },
      }
    );
    
    return response.data;
  } catch (error) {
    console.error('LWA token exchange failed:', error);
    return null;
  }
}
```

---

## Security Considerations

### Token Security

1. **JWT Secret**: Use a strong, randomly generated secret (256+ bits)
2. **Token Storage**: Store tokens encrypted at rest
3. **Token Transmission**: Always use HTTPS
4. **Token Revocation**: Implement immediate revocation on logout/unlink

### Client Security

1. **Client Secrets**: Store hashed, never in logs
2. **Redirect URI Validation**: Exact match only, no wildcards
3. **Rate Limiting**: Limit token requests to prevent brute force

### Session Security

1. **Session Cookies**: HttpOnly, Secure, SameSite=Strict
2. **CSRF Protection**: State parameter validation
3. **Session Expiry**: Short-lived sessions (15 minutes inactive)

### Password Security

1. **Hashing**: bcrypt with cost factor 12
2. **Requirements**: Minimum 8 characters, complexity check
3. **Rate Limiting**: Max 5 failed attempts, then lockout
