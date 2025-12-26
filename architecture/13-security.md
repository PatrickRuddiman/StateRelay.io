# 13 - Security Considerations

## Authentication Security

### Password Requirements
- Minimum 8 characters
- bcrypt hashing with cost factor 12
- Rate limit: 5 failed attempts → 15 minute lockout

### JWT Tokens
- Algorithm: HS256
- Access token: 1 hour expiry
- Secret: 256-bit randomly generated
- Store in httpOnly cookies for web

### OAuth 2.0
- Authorization code flow only (no implicit)
- State parameter required (CSRF protection)
- Exact redirect URI matching
- Client secrets stored hashed

## Data Security

### Multi-Tenancy Isolation
- Row-Level Security (RLS) on all tables
- Tenant ID checked on every query
- User can only access own devices

```sql
CREATE POLICY devices_user_isolation ON devices
    FOR ALL USING (user_id = current_setting('app.current_user_id')::UUID);
```

### Encryption
- TLS 1.2+ for all external traffic (Caddy)
- Database connections via SSL
- Sensitive tokens encrypted at rest

### Secrets Management
- All secrets via environment variables
- Never in code or logs
- GitHub Secrets for CI/CD
- Rotate secrets quarterly

## Network Security

### Caddy Configuration
```caddyfile
{
    servers {
        protocols h1 h2
    }
}

(security_headers) {
    header {
        Strict-Transport-Security "max-age=31536000; includeSubDomains"
        X-Content-Type-Options "nosniff"
        X-Frame-Options "DENY"
        X-XSS-Protection "1; mode=block"
        Referrer-Policy "strict-origin-when-cross-origin"
    }
}

api.staterelay.io {
    import security_headers
    reverse_proxy api:4000
}
```

### Rate Limiting
| Endpoint | Limit |
|----------|-------|
| `/oauth/token` | 10/minute/IP |
| `/api/*` | 100/minute/user |
| Alexa directives | 1000/minute/user |
| IFTTT triggers | 100/minute/user |

```typescript
import rateLimit from 'express-rate-limit';

const apiLimiter = rateLimit({
  windowMs: 60 * 1000, // 1 minute
  max: 100,
  keyGenerator: (req) => req.user?.id || req.ip,
});
```

### CORS
```typescript
app.use(cors({
  origin: ['https://app.staterelay.io'],
  credentials: true,
}));
```

## Input Validation

### All Inputs Validated
```typescript
import { z } from 'zod';

const CreateDeviceSchema = z.object({
  name: z.string().min(1).max(100).regex(/^[a-z0-9-]+$/),
  friendlyName: z.string().min(1).max(255),
  deviceType: z.enum(['switch', 'dimmer', 'sensor']),
});
```

### SQL Injection Prevention
- Prisma ORM with parameterized queries
- No raw SQL with user input

### XSS Prevention
- React auto-escapes output
- Content-Security-Policy headers

## Logging & Monitoring

### What to Log
- Authentication events (success/failure)
- Authorization failures
- Device state changes
- API errors

### What NOT to Log
- Passwords
- Tokens
- Personal data

### Log Format
```json
{
  "timestamp": "2025-01-15T10:30:00Z",
  "level": "info",
  "event": "auth_success",
  "userId": "uuid",
  "ip": "redacted"
}
```

## Dependency Security

### Keep Updated
```bash
# Check for vulnerabilities
npm audit

# Update dependencies
npm update
```

### Dependabot
Enable in `.github/dependabot.yml`:
```yaml
version: 2
updates:
  - package-ecosystem: npm
    directory: "/"
    schedule:
      interval: weekly
```

## Incident Response

1. **Detection**: Monitoring alerts
2. **Containment**: Rotate affected secrets
3. **Investigation**: Review logs
4. **Recovery**: Deploy fixes
5. **Post-mortem**: Document and improve

## Compliance Checklist

- [ ] TLS everywhere
- [ ] Passwords hashed
- [ ] Secrets not in code
- [ ] Input validation
- [ ] Rate limiting
- [ ] Audit logging
- [ ] Data isolation (RLS)
- [ ] Regular dependency updates
