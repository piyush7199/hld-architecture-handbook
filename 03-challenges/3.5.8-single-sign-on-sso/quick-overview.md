# Single Sign-On (SSO) System - Quick Overview

## Core Concept

Single Sign-On (SSO) enables users to authenticate once and access multiple applications without re-entering credentials. The SSO provider (like Okta, Auth0, or Azure AD) acts as a trusted identity broker, issuing tokens that applications validate to grant access.

**How It Works:**
- User logs in to SSO provider (username + password + MFA)
- SSO issues access token (JWT) containing user identity and permissions
- User accesses application → application validates token with SSO
- Application grants access based on token claims

## Requirements

### Functional Requirements
- OAuth 2.0 / OIDC support (modern applications)
- SAML 2.0 support (enterprise legacy systems)
- Social login (Google, Facebook, Microsoft, GitHub)
- Username/password authentication
- Multi-factor authentication (MFA): TOTP, SMS, push, hardware keys
- Session management (single sign-on, single sign-out)
- Token management (access tokens, refresh tokens, ID tokens)
- Identity federation (external providers: LDAP, Active Directory)
- Multi-tenancy (isolated identity data per organization)

### Non-Functional Requirements
- **High Availability**: 99.99% uptime (4 nines)
- **Low Latency**: Authentication <200ms (p95), token validation <50ms
- **Security**: End-to-end encryption, token signing, secure storage
- **Scalability**: 100M+ users, 10K+ applications, 1B+ auth requests/day
- **Multi-Tenancy**: Isolated data per organization (tenant)

## Components

### 1. Authentication API
- Handles login requests (username + password + MFA)
- Issues access tokens, refresh tokens, ID tokens
- Supports OAuth 2.0, OIDC, SAML 2.0 protocols
- Password hashing (bcrypt, cost factor 10)

### 2. Token Validation API
- Validates JWT access tokens (signature, expiration, claims)
- Token caching (Redis, 95% hit rate)
- Token blacklist (revoked tokens)
- **Highest QPS**: 50k QPS (peak)

### 3. Identity Database (PostgreSQL)
- User profiles, credentials, MFA settings
- Application registrations (client_id, client_secret)
- Refresh tokens (stateful, for revocation)
- Sharded by tenant_id (64 shards)

### 4. Token Cache (Redis)
- Access token → user mapping (fast validation)
- Authorization codes (10-minute TTL)
- Token blacklist (revoked tokens)
- Cache hit rate: 95% (critical for performance)

### 5. External Identity Providers
- Social login: Google, Facebook, Microsoft, GitHub, Apple
- Enterprise: LDAP, Active Directory, Azure AD
- Federation: Other SSO providers (Okta, Auth0)

## Architecture Flow

### OAuth 2.0 Authorization Code Flow

**Request Path:**
1. User → Application: "I want to log in"
2. Application → SSO: Redirect to /authorize?client_id=...&redirect_uri=...
3. SSO → User: Show login form
4. User → SSO: Enter credentials (username + password + MFA)
5. SSO → SSO: Validate credentials, generate authorization_code
6. SSO → Application: Redirect with authorization_code
7. Application → SSO: POST /token {code, client_secret}
8. SSO → Application: {access_token (JWT), refresh_token, id_token}

**Token Validation Path:**
1. Application → Resource: GET /api/data {Authorization: Bearer access_token}
2. Resource → SSO: Validate token (check signature, expiration)
3. SSO → Resource: Return user info (from JWT claims or cache)
4. Resource → Application: Return data

### SAML 2.0 Flow (Enterprise)

**Request Path:**
1. User → Application: Access protected resource
2. Application → SSO: Redirect to /saml/sso?SAMLRequest=... (XML)
3. SSO → User: Show login form (if not authenticated)
4. User → SSO: Enter credentials
5. SSO → SSO: Generate SAML assertion (XML)
6. SSO → Application: POST /saml/acs {SAMLResponse (XML)}
7. Application → Application: Parse SAML, create session
8. User → Application: Access granted

## Key Design Decisions

### 1. OAuth 2.0 vs SAML 2.0

**OAuth 2.0 / OIDC:**
- ✅ Modern standard, mobile support
- ✅ Stateless (JWT tokens)
- ✅ Fine-grained permissions (scopes)
- ❌ Not suitable for enterprise legacy systems

**SAML 2.0:**
- ✅ Enterprise standard (Active Directory integration)
- ✅ Single sign-out (SLO)
- ✅ Attribute-based authorization
- ❌ Complex (XML parsing), slower than OAuth

**Decision**: Support both (OAuth for modern apps, SAML for enterprise)

### 2. JWT vs Sessions

**JWT (Stateless):**
- ✅ No server-side storage (scales horizontally)
- ✅ Self-contained (includes user info)
- ❌ Can't revoke immediately (until expiration)

**Sessions (Stateful):**
- ✅ Can revoke immediately
- ✅ Server controls session lifecycle
- ❌ Requires shared session store (Redis)

**Decision**: Hybrid approach (JWT access tokens + stateful refresh tokens)

### 3. Token Lifetime

**Access Token:**
- Lifetime: 15 minutes (short-lived)
- Reason: Limits damage if stolen
- Trade-off: More refresh token requests

**Refresh Token:**
- Lifetime: 30 days (long-lived)
- Reason: Better user experience (fewer logins)
- Security: Token rotation (new refresh token on each use)

### 4. Multi-Tenancy

**Sharding Strategy:**
- Shard by tenant_id (hash(tenant_id) % 64)
- All tenant data on same shard (fast queries)
- Even distribution across shards

**Isolation:**
- Row-level security (PostgreSQL)
- Tenant context in all queries
- Separate client_id per tenant

## Bottlenecks & Solutions

### 1. Token Validation (50k QPS)

**Problem:** Token validation is highest QPS operation.

**Solution: JWT + Redis Cache**
- JWT validation: Check signature, expiration (<50ms)
- Redis cache: Token → user mapping (<1ms)
- Cache hit rate: 95% (most tokens validated multiple times)
- Average latency: <5ms (cache hit) vs <50ms (cache miss)

### 2. Password Hashing (CPU-Intensive)

**Problem:** bcrypt is slow (~100ms per hash), limits login throughput.

**Solution: Async Processing + Connection Pooling**
- Async password hashing (non-blocking)
- Connection pooling: 2,000 total connections
- Handles 1,750 QPS (avg 1.1ms per query)

### 3. Database Sharding

**Problem:** 100M users, single database can't handle load.

**Solution: Shard by Tenant ID**
- 64 shards (hash(tenant_id) % 64)
- 1,750 QPS / 64 shards = ~27 QPS per shard (well within limits)

## Common Anti-Patterns

### ❌ **1. Storing Passwords in Plaintext**

**Problem:**
```sql
CREATE TABLE users (password TEXT);  -- Plaintext!
```

**Solution:**
```sql
CREATE TABLE users (password_hash TEXT);  -- bcrypt hash
```

### ❌ **2. Long-Lived Access Tokens**

**Problem:**
```javascript
access_token = { "exp": now + 30_days }  // Too long!
```

**Solution:**
```javascript
access_token = { "exp": now + 15_minutes }  // Short-lived
// Use refresh token for long sessions
```

### ❌ **3. No Token Rotation**

**Problem:**
```javascript
refresh_token = "abc123"  // Never changes
```

**Solution:**
```javascript
// Refresh token rotation
old_refresh_token → new_access_token + new_refresh_token
old_refresh_token invalidated
```

### ❌ **4. Storing Tokens in localStorage**

**Problem:**
```javascript
localStorage.setItem("access_token", token)  // XSS risk!
```

**Solution:**
```javascript
// HttpOnly cookie
Set-Cookie: access_token=...; HttpOnly; Secure; SameSite=Strict
```

## Monitoring & Observability

### Key Metrics

**Authentication:**
- Login success rate (target: >99%)
- Login latency (p95: <200ms)
- MFA completion rate
- Failed login attempts (security)

**Token Validation:**
- Token validation QPS (peak: 50k QPS)
- Token validation latency (p95: <50ms)
- Token cache hit rate (target: >95%)
- Refresh token rotation rate

**Availability:**
- Uptime (target: 99.99%)
- Error rate (target: <0.1%)
- Database connection pool usage
- Redis cache hit rate

### Alerts

1. High error rate: >1% errors for 5 minutes
2. High latency: p95 > 500ms for 5 minutes
3. Database connection exhaustion: >80% pool usage
4. Redis failure: Cache unavailable
5. High failed login rate: >10% failed logins (potential attack)

## Trade-offs Summary

| What You Gain | What You Sacrifice |
|---------------|-------------------|
| ✅ **Single Sign-On**: Users log in once, access all apps | ❌ **Complexity**: Multiple protocols, token management |
| ✅ **Security**: Industry-standard protocols (OAuth, SAML) | ❌ **Latency**: Token validation adds ~50ms per request |
| ✅ **Scalability**: Stateless JWT tokens (horizontal scaling) | ❌ **Revocation**: Can't revoke JWT immediately (until expiration) |
| ✅ **Multi-Protocol**: OAuth, SAML, OIDC support | ❌ **Maintenance**: Multiple protocol implementations |
| ✅ **Identity Federation**: Connect with external providers | ❌ **Dependency**: Relies on external providers (Google, Microsoft) |
| ✅ **Multi-Tenancy**: Isolated data per organization | ❌ **Complexity**: Tenant isolation, data sharding |

## Real-World Examples

### Okta
- **Scale**: 15,000+ customers, 100M+ users
- **Protocols**: OAuth 2.0, SAML 2.0, LDAP
- **Features**: Multi-tenant SaaS, identity federation, MFA

### Auth0
- **Scale**: 7,000+ customers, 2.5B+ logins/month
- **Protocols**: OAuth 2.0 / OIDC
- **Features**: Social login (50+ providers), rules engine, multi-region

### Microsoft Azure AD
- **Scale**: 500M+ users
- **Protocols**: SAML 2.0 (enterprise), OAuth 2.0 / OIDC (modern)
- **Features**: Active Directory integration, conditional access, MFA

## Key Takeaways

1. **Multi-Protocol Support**: OAuth 2.0/OIDC for modern apps, SAML 2.0 for enterprise
2. **Stateless Tokens**: JWT access tokens (stateless, scalable)
3. **Token Lifecycle**: Short-lived access tokens (15 min), refresh token rotation
4. **Identity Federation**: Connect with external providers (Google, Microsoft, LDAP)
5. **Multi-Tenancy**: Isolated identity data per organization (sharding)
6. **High Availability**: 99.99% uptime, multi-region deployment
7. **Security**: Token signing, encryption, MFA, token blacklist
8. **Performance**: Token caching (Redis, 95% hit rate), database sharding
9. **Scalability**: Horizontal scaling (stateless services), 100M+ users
10. **Design Philosophy**: Security and scalability over simplicity

## Recommended Stack

**Authentication Service:**
- Language: Node.js / Python / Go
- Framework: Express / FastAPI / Gin
- Database: PostgreSQL (sharded by tenant_id)
- Cache: Redis Cluster (token validation, session storage)

**Token Management:**
- JWT Library: jsonwebtoken (Node.js), PyJWT (Python)
- Signing: RS256 (RSA) or HS256 (HMAC)
- Token Storage: HttpOnly cookies (refresh tokens), memory (access tokens)

**External Providers:**
- Social Login: Google OAuth, Facebook Login, Microsoft Identity Platform
- Enterprise: LDAP, Active Directory, Azure AD

**Infrastructure:**
- Load Balancer: AWS ALB / Azure Load Balancer
- Database: AWS RDS / Azure Database (PostgreSQL)
- Cache: AWS ElastiCache / Azure Cache (Redis)
- CDN: CloudFront / Azure CDN (static assets)

