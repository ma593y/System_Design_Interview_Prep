# Security

## Overview

Security is critical in system design. A secure system protects data confidentiality, integrity, and availability.

## Security Principles

### CIA Triad

```
        Confidentiality
              /\
             /  \
            /    \
           /      \
          /________\
         /          \
        /            \
   Integrity ──── Availability
```

| Principle | Description | Example |
|-----------|-------------|---------|
| **Confidentiality** | Data accessible only to authorized users | Encryption, access control |
| **Integrity** | Data is accurate and unaltered | Checksums, digital signatures |
| **Availability** | System is accessible when needed | Redundancy, DDoS protection |

### Defense in Depth

Multiple layers of security.

```
┌─────────────────────────────────────────────────────────┐
│                     Perimeter                            │
│   ┌─────────────────────────────────────────────────┐   │
│   │                   Network                        │   │
│   │   ┌─────────────────────────────────────────┐   │   │
│   │   │              Application                 │   │   │
│   │   │   ┌─────────────────────────────────┐   │   │   │
│   │   │   │            Data                  │   │   │   │
│   │   │   └─────────────────────────────────┘   │   │   │
│   │   └─────────────────────────────────────────┘   │   │
│   └─────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

### Least Privilege

Grant minimum permissions necessary.

```
User Roles:
- Admin: Full access
- Editor: Read + Write
- Viewer: Read only
- Guest: Public data only
```

## Authentication

Verifying identity (who are you?).

### Password Authentication

```
Best Practices:
- Hash passwords (bcrypt, Argon2)
- Salt each password
- Enforce strong password policies
- Implement account lockout
- Support 2FA

Storage:
Password → Salt + Hash → Database
  "secret"    "$2b$10$xyz..."
```

### Multi-Factor Authentication (MFA)

```
Something you:
- Know: Password, PIN
- Have: Phone, hardware token
- Are: Fingerprint, face

Common flows:
1. Password + SMS code
2. Password + Authenticator app (TOTP)
3. Password + Hardware key (FIDO2)
```

### Token-Based Authentication

#### Session Tokens

```
Login → Server creates session → Returns session ID
          ↓
    Stored in server memory/Redis
          ↓
Client stores in cookie, sends with each request
```

#### JWT (JSON Web Tokens)

```
Structure: Header.Payload.Signature

Header: {"alg": "HS256", "typ": "JWT"}
Payload: {"sub": "user123", "exp": 1642000000, "roles": ["user"]}
Signature: HMACSHA256(header + payload, secret)

Stateless: Server doesn't store session
Verify: Check signature + expiration
```

**JWT Best Practices:**
- Short expiration (15 min - 1 hour)
- Use refresh tokens for renewal
- Store in httpOnly cookies or memory
- Include only necessary claims

### OAuth 2.0

Delegated authorization framework.

```
Authorization Code Flow:

┌──────┐     ┌────────────┐     ┌─────────────┐
│ User │     │   Client   │     │ Auth Server │
└──┬───┘     └─────┬──────┘     └──────┬──────┘
   │               │                    │
   │──── Login ───▶│                    │
   │               │── Auth Request ───▶│
   │◀───────────── Redirect to Login ───│
   │──────────── Authenticate ─────────▶│
   │◀───────────── Auth Code ───────────│
   │               │◀── Auth Code ──────│
   │               │── Token Request ──▶│
   │               │◀─ Access Token ────│
   │               │                    │
   │◀─ Protected ──│                    │
   │   Resource    │                    │
```

**OAuth 2.0 Grant Types:**
- Authorization Code: Web apps with backend
- PKCE: Mobile/SPA apps
- Client Credentials: Service-to-service
- Refresh Token: Long-lived access

### OpenID Connect (OIDC)

Identity layer on top of OAuth 2.0.

```
OAuth 2.0: Authorization (what can you access?)
OIDC: Authentication (who are you?)

Adds:
- ID Token (JWT with user info)
- UserInfo endpoint
- Standard scopes (openid, profile, email)
```

## Authorization

Controlling access (what can you do?).

### RBAC (Role-Based Access Control)

```
Users → Roles → Permissions

User: Alice
  └─ Role: Editor
       └─ Permissions: read, write, delete_own

User: Bob
  └─ Role: Viewer
       └─ Permissions: read
```

### ABAC (Attribute-Based Access Control)

```
Policy: Allow if (user.department == resource.department
                  AND user.clearance >= resource.classification)

More flexible but complex
```

### Resource-Based Authorization

```python
# Check ownership
def can_edit(user, document):
    return document.owner_id == user.id or user.is_admin

# Check permissions
def can_delete(user, document):
    return user.has_permission('document.delete')
           and document.owner_id == user.id
```

## Encryption

### Encryption at Rest

Protecting stored data.

```
Application → Encrypt → Storage
     ↓
    Key Management Service
     ↓
   Master Key (HSM)

Types:
- Full disk encryption
- Database encryption
- File-level encryption
```

### Encryption in Transit

Protecting data during transmission.

```
TLS/HTTPS:

Client ──────── TLS Handshake ──────── Server
       ◀──────────────────────────────▶
         Encrypted communication

TLS 1.3:
- Faster handshake (1-RTT)
- Perfect forward secrecy
- Modern cipher suites
```

### End-to-End Encryption (E2EE)

Data encrypted from sender to recipient.

```
Sender → Encrypt with recipient's public key → Server → Decrypt with private key → Recipient

Server cannot read the data
```

## Common Vulnerabilities (OWASP Top 10)

### SQL Injection

```sql
# Vulnerable
query = f"SELECT * FROM users WHERE id = {user_input}"
# Input: "1 OR 1=1" → Returns all users

# Safe: Parameterized queries
cursor.execute("SELECT * FROM users WHERE id = ?", [user_input])
```

### XSS (Cross-Site Scripting)

```html
# Vulnerable
<div>${userInput}</div>
# Input: "<script>stealCookies()</script>"

# Safe: Escape output
<div>${escapeHtml(userInput)}</div>
```

### CSRF (Cross-Site Request Forgery)

```
Attack: Malicious site triggers action on trusted site
Defense:
- CSRF tokens
- SameSite cookies
- Verify Origin/Referer headers
```

### Broken Authentication

```
Vulnerabilities:
- Weak passwords allowed
- No brute force protection
- Session doesn't expire
- Predictable session IDs

Defenses:
- Strong password policy
- Rate limiting
- Secure session management
- MFA
```

## Security Headers

```
Content-Security-Policy: default-src 'self'
Strict-Transport-Security: max-age=31536000; includeSubDomains
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
X-XSS-Protection: 1; mode=block
Referrer-Policy: strict-origin-when-cross-origin
```

## API Security

### Rate Limiting

```
Prevent:
- Brute force attacks
- DoS attacks
- Resource exhaustion

Implementation:
- Token bucket
- Sliding window
- Per-user, per-IP limits
```

### Input Validation

```python
# Validate all input
def create_user(data):
    # Type checking
    assert isinstance(data['name'], str)
    assert isinstance(data['age'], int)

    # Length limits
    assert len(data['name']) <= 100

    # Format validation
    assert re.match(r'^[\w@.]+$', data['email'])

    # Range validation
    assert 0 <= data['age'] <= 150
```

### API Keys

```
Best Practices:
- Use strong random keys
- Hash stored keys
- Rotate regularly
- Scope permissions
- Log usage
- Revoke on breach
```

## Secrets Management

```
Don't:
- Hardcode secrets
- Commit to version control
- Log secrets
- Share via insecure channels

Do:
- Use secrets managers (Vault, AWS Secrets Manager)
- Environment variables for local dev
- Rotate secrets regularly
- Audit access
```

## Security Monitoring

### Logging

```
Log:
- Authentication events
- Authorization failures
- Input validation errors
- System access
- Data modifications

Don't log:
- Passwords
- Full credit card numbers
- Personal health information
- Session tokens
```

### Intrusion Detection

```
Monitor for:
- Unusual login patterns
- Multiple failed attempts
- Geographic anomalies
- Privilege escalation
- Data exfiltration patterns
```

## Interview Talking Points

1. "Defense in depth - multiple layers of security"
2. "Never store passwords in plaintext - use bcrypt or Argon2"
3. "HTTPS everywhere, encrypt sensitive data at rest"
4. "Validate all input, escape all output"
5. "Least privilege - minimum necessary permissions"

## Common Interview Questions

1. **Q: How would you secure user passwords?**
   A: Hash with bcrypt/Argon2, unique salt per password, enforce strong passwords, implement MFA.

2. **Q: How do you prevent SQL injection?**
   A: Use parameterized queries/prepared statements. Never concatenate user input into queries.

3. **Q: JWT vs Session tokens?**
   A: JWT is stateless (scalable) but can't be invalidated before expiry. Sessions can be invalidated but require server-side storage.

4. **Q: How do you handle secrets in production?**
   A: Use secrets managers (Vault, AWS Secrets Manager), never in code or version control, rotate regularly.

## Key Takeaways

- Security must be designed in, not added later
- Use proven libraries and frameworks
- Keep software updated
- Monitor and log security events
- Plan for breach response

## Further Reading

- OWASP Top 10
- "The Web Application Hacker's Handbook"
- NIST Cybersecurity Framework
- OAuth 2.0 and OpenID Connect specifications
