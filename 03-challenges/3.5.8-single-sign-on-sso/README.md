# 3.5.8 Design Single Sign-On (SSO) System (Okta / Auth0 / Microsoft Azure AD)

> 📚 **Note on Implementation Details:**
> This document focuses on high-level design concepts and architectural decisions.
> For detailed algorithm implementations, see **[pseudocode.md](./pseudocode.md)**.

## 📊 Visual Diagrams & Resources

- **[High-Level Design Diagrams](./hld-diagram.md)** - System architecture, component design, data flow
- **[Sequence Diagrams](./sequence-diagrams.md)** - Detailed interaction flows and failure scenarios
- **[Design Decisions (This Over That)](./this-over-that.md)** - In-depth analysis of architectural choices
- **[Pseudocode Implementations](./pseudocode.md)** - Detailed algorithm implementations

---

## 1. Problem Statement

Design a **Single Sign-On (SSO) system** like Okta, Auth0, or Microsoft Azure AD that enables users to authenticate once and access multiple applications without re-entering credentials. The system must support **OAuth 2.0/OIDC**, **SAML 2.0**, **social login** (Google, Facebook), **multi-factor authentication (MFA)**, **session management**, and **identity federation** across thousands of applications.

**Key Challenges:**

- **Multiple Protocols**: Support OAuth 2.0, OIDC, SAML 2.0 simultaneously
- **Session Management**: Share sessions across multiple applications (stateless vs stateful)
- **Token Lifecycle**: Access tokens, refresh tokens, ID tokens with proper expiration and rotation
- **Identity Federation**: Connect with external identity providers (Google, Microsoft, LDAP)
- **Multi-Tenancy**: Support thousands of organizations (tenants) with isolated identity data
- **High Availability**: 99.99% uptime (authentication is critical path)
- **Security**: Prevent token theft, replay attacks, session hijacking
- **Scalability**: Handle millions of users, billions of authentication requests

**Real-World Examples:**

- **Okta**: 15,000+ customers, 100M+ users, supports OAuth, SAML, LDAP
- **Auth0**: 7,000+ customers, handles 2.5B+ logins/month
- **Microsoft Azure AD**: 500M+ users, enterprise SSO, MFA
- **Google Workspace SSO**: 5M+ organizations, OAuth 2.0, SAML

**The Core Challenge:**

Traditional authentication requires users to log in to each application separately. SSO must:

1. **Authenticate once**: User logs in to SSO provider
2. **Share session**: Multiple applications trust SSO provider
3. **Support multiple protocols**: OAuth, SAML, OpenID Connect
4. **Federate identities**: Connect with external providers
5. **Scale globally**: Millions of users, thousands of applications

---

## 2. Requirements and Scale Estimation

### Functional Requirements (FRs)

1. **OAuth 2.0/OIDC Support**: Authorization Code Flow, Implicit Flow, Client Credentials
2. **SAML 2.0 Support**: Enterprise SSO with XML-based assertions
3. **Social Login**: Google, Facebook, Microsoft, GitHub, Apple
4. **Username/Password**: Traditional authentication with password hashing
5. **Multi-Factor Authentication (MFA)**: TOTP, SMS, push notifications, hardware keys
6. **Session Management**: Single sign-on, single sign-out, session sharing
7. **Token Management**: Access tokens, refresh tokens, ID tokens with rotation
8. **Identity Federation**: Connect with external identity providers (LDAP, Active Directory)
9. **User Management**: User provisioning, deprovisioning, profile management
10. **Application Management**: Register applications, configure SSO settings

### Non-Functional Requirements (NFRs)

1. **High Availability**: 99.99% uptime (4 nines = 52 minutes downtime/year)
2. **Low Latency**: Authentication < 200ms (p95), token validation < 50ms
3. **Security**: End-to-end encryption, token signing, secure storage
4. **Scalability**: Support 100M+ users, 10K+ applications, 1B+ auth requests/day
5. **Multi-Tenancy**: Isolated identity data per organization
6. **Compliance**: SOC 2, GDPR, HIPAA, PCI-DSS (for payment apps)

### Scale Estimation

| Metric | Assumption | Calculation | Result |
|--------|------------|-------------|--------|
| **Total Users** | Global adoption | - | 100 million users |
| **Active Users** | Daily usage | 30% of total | 30M daily active users |
| **Authentication Requests** | Per user per day | 30M × 5 logins/day | 150M auth requests/day |
| **Auth QPS** | Peak load | $\frac{150 \text{M}}{86400 \text{s}}$ | ~1,750 QPS (peak: 5k QPS) |
| **Token Validations** | Per request | 150M × 10 validations | 1.5B validations/day |
| **Token Validation QPS** | Peak load | $\frac{1.5 \text{B}}{86400 \text{s}}$ | ~17k QPS (peak: 50k QPS) |
| **Applications** | Per organization | 10K organizations × 10 apps | 100K applications |
| **Organizations (Tenants)** | Multi-tenant | - | 10,000 organizations |
| **Storage per User** | Profile + tokens | 5 KB/user | 500 GB total |
| **Token Storage** | Active sessions | 30M × 2 KB | 60 GB (Redis) |

**Key Insight**: Token validation is the highest QPS operation (50k QPS peak). Must be optimized for low latency (<50ms).

---

## 3. High-Level Architecture

> 📊 **See detailed architecture:** [High-Level Design Diagrams](./hld-diagram.md)

The architecture follows a **stateless token-based design** with **OAuth 2.0/OIDC** for modern applications and **SAML 2.0** for enterprise legacy systems.

### Core Components

```
┌──────────────────────────────────────────────────────────────────┐
│                         Applications                              │
│              (Web Apps, Mobile Apps, APIs)                        │
└──────────────────────┬───────────────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────────────┐
│                      API Gateway / Load Balancer                  │
│              Rate Limiting • Routing • TLS Termination            │
└──────────────────────┬───────────────────────────────────────────┘
                       │
        ┌──────────────┼──────────────┐
        │              │              │
        ▼              ▼              ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│  Auth API    │ │  Token API   │ │  User API    │
│  (Login)     │ │  (Validate)  │ │  (Profile)   │
└──────┬───────┘ └──────┬───────┘ └──────┬───────┘
       │                │                 │
       │                ▼                 │
       │        ┌──────────────┐          │
       │        │ Token Cache  │          │
       │        │   (Redis)    │          │
       │        └──────────────┘          │
       │                                  │
       └──────────────┬───────────────────┘
                      │
                      ▼
            ┌──────────────────────┐
            │   Identity Database  │
            │   (PostgreSQL)        │
            │   Sharded by Tenant   │
            └──────────────────────┘
                      │
                      ▼
            ┌──────────────────────┐
            │  External Providers  │
            │  (Google, Microsoft,  │
            │   LDAP, Active Dir)   │
            └──────────────────────┘
```

### Key Design Principles

1. **Stateless Tokens**: JWT access tokens (no server-side session storage)
2. **Token Caching**: Redis for fast token validation (50k QPS)
3. **Multi-Protocol**: OAuth 2.0, OIDC, SAML 2.0 support
4. **Identity Federation**: Connect with external providers
5. **Multi-Tenancy**: Isolated data per organization (tenant)
6. **Horizontal Scaling**: Stateless services (scale independently)

---

## 4. Detailed Component Design

### 4.1 OAuth 2.0 / OIDC Flow

**Why OAuth 2.0?**

- Industry standard for modern applications
- Stateless (JWT tokens)
- Fine-grained permissions (scopes)
- Mobile and web support

**Authorization Code Flow (Most Secure):**

```
1. User → Application: "I want to log in"
2. Application → SSO: Redirect to /authorize?client_id=...&redirect_uri=...
3. SSO → User: Show login form
4. User → SSO: Enter credentials (username + password + MFA)
5. SSO → Application: Redirect with authorization_code
6. Application → SSO: POST /token {code, client_secret}
7. SSO → Application: {access_token, refresh_token, id_token}
8. Application → Resource: GET /api/data {Authorization: Bearer access_token}
9. Resource → SSO: Validate token (check signature, expiration)
10. Resource → Application: Return data
```

*See pseudocode.md::oauth_authorization_code_flow() for implementation*

### 4.2 SAML 2.0 Flow (Enterprise)

**Why SAML 2.0?**

- Enterprise standard (legacy systems)
- XML-based assertions
- Single sign-out (SLO)
- Attribute-based authorization

**SAML Flow:**

```
1. User → Application: Access protected resource
2. Application → SSO: Redirect to /saml/sso?SAMLRequest=...
3. SSO → User: Show login form (if not authenticated)
4. User → SSO: Enter credentials
5. SSO → Application: POST /saml/acs {SAMLResponse (XML)}
6. Application → Application: Parse SAML, create session
7. User → Application: Access granted
```

*See pseudocode.md::saml_sso_flow() for implementation*

### 4.3 Token Management

**Token Types:**

1. **Access Token (JWT)**: Short-lived (15 minutes), contains user info and permissions
2. **Refresh Token**: Long-lived (30 days), used to get new access tokens
3. **ID Token (JWT)**: Contains user identity (OIDC), short-lived (15 minutes)

**Token Storage:**

- **Access Token**: Client-side (HttpOnly cookie or memory)
- **Refresh Token**: HttpOnly cookie (more secure) or encrypted database
- **Token Blacklist**: Redis for revoked tokens (TTL = token expiration)

**Token Rotation:**

```
1. Client uses refresh_token to get new access_token
2. Server issues new refresh_token
3. Old refresh_token invalidated
4. Benefits: Detects token theft, limits damage
```

*See pseudocode.md::refresh_access_token() for implementation*

### 4.4 Session Management

**Stateless vs Stateful:**

**Stateless (JWT-based):**
- Access token contains all user info
- No server-side session storage
- Scales horizontally (no shared state)
- Trade-off: Can't revoke immediately (until expiration)

**Stateful (Session-based):**
- Server stores session in Redis
- Session ID in cookie
- Can revoke immediately
- Trade-off: Requires shared session store

**Hybrid Approach (Recommended):**

```
- Access token: JWT (stateless, 15 minutes)
- Refresh token: Stateful (Redis, 30 days)
- Benefits: Fast validation (JWT), immediate revocation (refresh token)
```

*See pseudocode.md::create_session() and pseudocode.md::validate_session() for implementation*

### 4.5 Identity Federation

**External Identity Providers:**

1. **Social Login**: Google, Facebook, Microsoft, GitHub, Apple
2. **Enterprise**: LDAP, Active Directory, Azure AD
3. **Other SSO Providers**: Okta, Auth0 (federation)

**Federation Flow:**

```
1. User → SSO: "Login with Google"
2. SSO → Google: OAuth redirect
3. Google → User: Consent screen
4. Google → SSO: Authorization code
5. SSO → Google: Exchange code for user info
6. SSO → SSO: Create/link local account
7. SSO → Application: Issue access token
```

*See pseudocode.md::federate_identity() for implementation*

### 4.6 Multi-Factor Authentication (MFA)

**MFA Methods:**

1. **TOTP**: Time-based one-time password (Google Authenticator)
2. **SMS OTP**: One-time code via SMS
3. **Push Notification**: Approve via mobile app
4. **Hardware Keys**: FIDO2/WebAuthn (YubiKey)

**MFA Flow:**

```
1. User → SSO: Enter username + password
2. SSO → SSO: Validate credentials
3. SSO → User: "Enter 6-digit code from authenticator"
4. User → SSO: Enter TOTP code
5. SSO → SSO: Validate TOTP
6. SSO → Application: Issue access token
```

*See pseudocode.md::verify_mfa() for implementation*

---

## 5. Data Models

### 5.1 User Schema

```sql
CREATE TABLE users (
    user_id UUID PRIMARY KEY,
    tenant_id UUID NOT NULL,  -- Multi-tenancy
    email TEXT UNIQUE NOT NULL,
    username TEXT UNIQUE,
    password_hash TEXT,  -- bcrypt, nullable (social login users)
    email_verified BOOLEAN DEFAULT FALSE,
    mfa_enabled BOOLEAN DEFAULT FALSE,
    mfa_secret TEXT,  -- TOTP secret (encrypted)
    created_at TIMESTAMP NOT NULL,
    updated_at TIMESTAMP NOT NULL,
    last_login TIMESTAMP,
    FOREIGN KEY (tenant_id) REFERENCES tenants(tenant_id)
);

CREATE INDEX idx_tenant_email ON users(tenant_id, email);
CREATE INDEX idx_email ON users(email);
```

### 5.2 Application Schema

```sql
CREATE TABLE applications (
    app_id UUID PRIMARY KEY,
    tenant_id UUID NOT NULL,
    client_id TEXT UNIQUE NOT NULL,
    client_secret TEXT NOT NULL,  -- Hashed
    name TEXT NOT NULL,
    redirect_uris TEXT[] NOT NULL,  -- Allowed redirect URIs
    protocol TEXT NOT NULL,  -- 'oauth2', 'oidc', 'saml'
    scopes TEXT[],  -- OAuth scopes
    created_at TIMESTAMP NOT NULL,
    FOREIGN KEY (tenant_id) REFERENCES tenants(tenant_id)
);

CREATE INDEX idx_client_id ON applications(client_id);
```

### 5.3 Token Schema

```sql
CREATE TABLE refresh_tokens (
    token_id UUID PRIMARY KEY,
    user_id UUID NOT NULL,
    app_id UUID NOT NULL,
    token_hash TEXT UNIQUE NOT NULL,  -- Hashed refresh token
    expires_at TIMESTAMP NOT NULL,
    revoked BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP NOT NULL,
    FOREIGN KEY (user_id) REFERENCES users(user_id),
    FOREIGN KEY (app_id) REFERENCES applications(app_id)
);

CREATE INDEX idx_user_app ON refresh_tokens(user_id, app_id);
CREATE INDEX idx_expires ON refresh_tokens(expires_at);
```

### 5.4 Session Schema (Stateful Sessions)

```sql
CREATE TABLE sessions (
    session_id UUID PRIMARY KEY,
    user_id UUID NOT NULL,
    tenant_id UUID NOT NULL,
    access_token_hash TEXT,  -- Hashed access token (if stateful)
    expires_at TIMESTAMP NOT NULL,
    created_at TIMESTAMP NOT NULL,
    FOREIGN KEY (user_id) REFERENCES users(user_id)
);

CREATE INDEX idx_user_session ON sessions(user_id);
CREATE INDEX idx_expires ON sessions(expires_at);
```

---

## 6. Key Algorithms

### 6.1 OAuth 2.0 Authorization Code Flow

**Steps:**

1. Generate authorization code (random, 32 bytes)
2. Store code → user mapping (Redis, 10-minute TTL)
3. Redirect user to application with code
4. Application exchanges code for tokens
5. Issue access token (JWT), refresh token, ID token

*See pseudocode.md::oauth_authorization_code_flow() for detailed implementation*

### 6.2 Token Validation

**JWT Validation:**

1. Verify signature (HMAC-SHA256 or RSA)
2. Check expiration (exp claim)
3. Check issuer (iss claim)
4. Check audience (aud claim)
5. Optional: Check token blacklist (Redis)

*See pseudocode.md::validate_access_token() for implementation*

### 6.3 Password Hashing

**bcrypt with Salt:**

```
password_hash = bcrypt.hash(password, cost_factor=10)
// Cost factor 10 = 2^10 = 1,024 iterations
// ~100ms on modern hardware
```

*See pseudocode.md::hash_password() and pseudocode.md::verify_password() for implementation*

---

## 7. Availability and Fault Tolerance

### 7.1 High Availability

**Multi-Region Deployment:**

- **Primary Region**: US-East (handles 70% of traffic)
- **Secondary Region**: EU-West (handles 20% of traffic)
- **Tertiary Region**: AP-South (handles 10% of traffic)

**Failover Strategy:**

- **Active-Active**: All regions serve traffic (DNS-based routing)
- **Database Replication**: Primary → Replicas (async, <1s lag)
- **Token Cache**: Redis Cluster (multi-region replication)

### 7.2 Failure Scenarios

**1. Database Failure:**

- **Impact**: Can't create new sessions, can't validate refresh tokens
- **Mitigation**: Read replicas (failover in <30s), token cache (continues working)

**2. Redis Failure:**

- **Impact**: Token validation slower (database lookup), can't revoke tokens
- **Mitigation**: Redis Cluster (automatic failover), fallback to database

**3. External Provider Failure (Google, Microsoft):**

- **Impact**: Social login unavailable
- **Mitigation**: Fallback to username/password, show error message

---

## 8. Bottlenecks and Optimizations

### 8.1 Token Validation (50k QPS)

**Problem:** Token validation is the highest QPS operation.

**Solution: JWT + Redis Cache**

```
Token Validation Flow:
  1. Check Redis cache (token → user mapping)
  2. If cache hit → return user info (<1ms)
  3. If cache miss → validate JWT signature
  4. Store in cache (TTL = token expiration)
  5. Return user info

Cache Hit Rate: 95% (most tokens validated multiple times)
Average Latency: <5ms (cache hit) vs <50ms (cache miss)
```

### 8.2 Password Hashing (CPU-Intensive)

**Problem:** bcrypt is slow (~100ms per hash), limits login throughput.

**Solution: Async Processing + Connection Pooling**

```
Login Flow:
  1. Receive login request
  2. Check rate limiting (Redis)
  3. Async: Hash password (non-blocking)
  4. Validate password hash
  5. Issue tokens

Connection Pooling:
  - PostgreSQL: 200 connections per instance
  - 10 instances × 200 = 2,000 total connections
  - Handles 1,750 QPS (avg 1.1ms per query)
```

### 8.3 Database Sharding

**Problem:** 100M users, single database can't handle load.

**Solution: Shard by Tenant ID**

```
Sharding Strategy:
  shard_id = hash(tenant_id) % 64
  
Benefits:
  - All tenant data on same shard (fast queries)
  - Even distribution across shards
  - 1,750 QPS / 64 shards = ~27 QPS per shard (well within limits)
```

---

## 9. Common Anti-Patterns

### ❌ **1. Storing Passwords in Plaintext**

**Problem:**
```sql
-- BAD: Plaintext passwords
CREATE TABLE users (
    password TEXT  -- Plaintext!
);
```

**Why It's Bad:**
- Database breach → attacker sees all passwords
- No security if database is compromised

**Solution:**
```sql
-- GOOD: Hashed with bcrypt
CREATE TABLE users (
    password_hash TEXT  -- bcrypt hash
);
```

### ❌ **2. Long-Lived Access Tokens**

**Problem:**
```
// BAD: Access token valid for 30 days
access_token = {
  "exp": now + 30_days  // Too long!
}
```

**Why It's Bad:**
- If stolen, attacker has access for 30 days
- Can't revoke immediately

**Solution:**
```
// GOOD: Short-lived access token (15 minutes)
access_token = {
  "exp": now + 15_minutes
}
// Use refresh token for long sessions
```

### ❌ **3. No Token Rotation**

**Problem:**
```
// BAD: Same refresh token reused
refresh_token = "abc123"  // Never changes
```

**Why It's Bad:**
- If stolen, attacker can use indefinitely
- No way to detect token theft

**Solution:**
```
// GOOD: Refresh token rotation
old_refresh_token → new_access_token + new_refresh_token
old_refresh_token invalidated
```

### ❌ **4. Storing Tokens in localStorage**

**Problem:**
```javascript
// BAD: localStorage accessible to JavaScript
localStorage.setItem("access_token", token)
// XSS attack → attacker steals token
```

**Why It's Bad:**
- XSS vulnerability → token stolen
- No protection against JavaScript access

**Solution:**
```javascript
// GOOD: HttpOnly cookie
Set-Cookie: access_token=...; HttpOnly; Secure; SameSite=Strict
// Not accessible to JavaScript
```

---

## 10. Alternative Approaches

### 10.1 Session-Based Authentication (Alternative to JWT)

**How It Works:**
- Server creates session (stored in Redis)
- Session ID in HttpOnly cookie
- Server validates session on each request

**Pros:**
- Can revoke immediately
- Server controls session lifecycle

**Cons:**
- Requires shared session store (Redis)
- Doesn't scale as well as JWT (stateless)

**When to Use:**
- Need immediate revocation
- Single-region deployment
- Traditional web applications

### 10.2 SAML-Only SSO (Enterprise)

**How It Works:**
- XML-based assertions
- Enterprise identity providers (Active Directory)
- Single sign-out (SLO)

**Pros:**
- Enterprise standard
- Mature, battle-tested
- Attribute-based authorization

**Cons:**
- Complex (XML parsing)
- Not suitable for mobile apps
- Slower than OAuth (XML overhead)

**When to Use:**
- Enterprise customers
- Legacy systems
- Active Directory integration

---

## 11. Monitoring and Observability

### 11.1 Key Metrics

**Authentication Metrics:**
- Login success rate (target: >99%)
- Login latency (p95: <200ms)
- MFA completion rate
- Failed login attempts (security)

**Token Metrics:**
- Token validation QPS (peak: 50k QPS)
- Token validation latency (p95: <50ms)
- Token cache hit rate (target: >95%)
- Refresh token rotation rate

**Availability Metrics:**
- Uptime (target: 99.99%)
- Error rate (target: <0.1%)
- Database connection pool usage
- Redis cache hit rate

### 11.2 Alerts

1. **High Error Rate**: >1% errors for 5 minutes
2. **High Latency**: p95 > 500ms for 5 minutes
3. **Database Connection Exhaustion**: >80% pool usage
4. **Redis Failure**: Cache unavailable
5. **High Failed Login Rate**: >10% failed logins (potential attack)

---

## 12. Trade-offs Summary

| What You Gain | What You Sacrifice |
|---------------|-------------------|
| ✅ **Single Sign-On**: Users log in once, access all apps | ❌ **Complexity**: Multiple protocols, token management |
| ✅ **Security**: Industry-standard protocols (OAuth, SAML) | ❌ **Latency**: Token validation adds ~50ms per request |
| ✅ **Scalability**: Stateless JWT tokens (horizontal scaling) | ❌ **Revocation**: Can't revoke JWT immediately (until expiration) |
| ✅ **Multi-Protocol**: OAuth, SAML, OIDC support | ❌ **Maintenance**: Multiple protocol implementations |
| ✅ **Identity Federation**: Connect with external providers | ❌ **Dependency**: Relies on external providers (Google, Microsoft) |
| ✅ **Multi-Tenancy**: Isolated data per organization | ❌ **Complexity**: Tenant isolation, data sharding |

**Design Philosophy**: **Security and scalability over simplicity** - multiple protocols, token management, and identity federation prioritize security and enterprise requirements, even if it means more complexity.

---

## 13. Real-World Examples

### Okta

**Scale:**
- 15,000+ customers
- 100M+ users
- Supports OAuth 2.0, SAML 2.0, LDAP

**Architecture:**
- Multi-tenant SaaS
- JWT access tokens
- Identity federation
- MFA support

### Auth0

**Scale:**
- 7,000+ customers
- 2.5B+ logins/month
- Global deployment

**Architecture:**
- OAuth 2.0 / OIDC
- Social login (50+ providers)
- Rules engine (custom logic)
- Multi-region

### Microsoft Azure AD

**Scale:**
- 500M+ users
- Enterprise SSO
- Active Directory integration

**Architecture:**
- SAML 2.0 (enterprise)
- OAuth 2.0 / OIDC (modern apps)
- Conditional access policies
- MFA (TOTP, SMS, push)

---

## 14. Deployment and Infrastructure

### 14.1 Multi-Region Deployment

**Regions:**
- **US-East**: Primary (70% traffic)
- **EU-West**: Secondary (20% traffic)
- **AP-South**: Tertiary (10% traffic)

**Components:**
- **API Services**: Auto-scaling (2-100 instances per region)
- **Database**: PostgreSQL (primary + read replicas)
- **Cache**: Redis Cluster (multi-region replication)
- **Load Balancer**: Global load balancer (route to nearest region)

### 14.2 Infrastructure as Code

**Terraform Configuration:**
- VPC, subnets, security groups
- RDS PostgreSQL (multi-AZ)
- ElastiCache Redis (cluster mode)
- Auto Scaling Groups
- Load Balancers

---

## 15. Advanced Features

### 15.1 Conditional Access

**Policies:**
- Require MFA for admin users
- Block access from certain countries
- Require device compliance (MDM)
- Time-based access (business hours only)

### 15.2 Just-In-Time (JIT) Provisioning

**How It Works:**
- User logs in via federation (Google, Microsoft)
- SSO automatically creates user account
- No manual user provisioning needed

### 15.3 Passwordless Authentication

**Methods:**
- Magic links (email)
- WebAuthn / FIDO2 (hardware keys)
- Biometric authentication

---

## 16. Interview Discussion Points

### Key Talking Points

1. **Why OAuth 2.0 over SAML?**
   - Modern standard, mobile support, stateless (JWT)
   - SAML is enterprise-only, XML overhead

2. **JWT vs Sessions?**
   - JWT: Stateless, scalable, but can't revoke immediately
   - Sessions: Can revoke, but requires shared store

3. **How to handle token theft?**
   - Short-lived access tokens (15 minutes)
   - Refresh token rotation
   - Token blacklist (Redis)
   - Device fingerprinting

4. **Multi-tenancy isolation?**
   - Shard by tenant_id
   - Row-level security (PostgreSQL)
   - Tenant context in all queries

5. **How to scale to 1B users?**
   - Database sharding (64 shards)
   - Token caching (Redis, 95% hit rate)
   - Horizontal scaling (stateless services)
   - CDN for static assets

---

## 17. References

### Related System Design Components

- **[2.4.4 OAuth & JWT Deep Dive](../02-components/2.4-security-observability/2.4.4-oauth-jwt-deep-dive.md)** - OAuth 2.0, JWT tokens, OpenID Connect
- **[2.4.1 Security Fundamentals](../02-components/2.4-security-observability/2.4.1-security-fundamentals.md)** - Authentication, authorization, encryption
- **[3.5.7 Authenticator App](./3.5.7-authenticator-app/README.md)** - MFA, TOTP, push notifications

### Related Design Challenges

- **[3.5.1 Payment Gateway](./3.5.1-payment-gateway/README.md)** - Security, encryption, compliance
- **[3.2.2 Notification Service](./3.2.2-notification-service/README.md)** - Push notifications for MFA

### External Resources

- **RFC 6749**: OAuth 2.0 Authorization Framework
- **RFC 7519**: JSON Web Token (JWT)
- **SAML 2.0**: Security Assertion Markup Language
- **OpenID Connect**: OIDC specification

### Books

- **"OAuth 2.0 in Action"** by Justin Richer, Antonio Sanso
- **"Identity and Data Security for Web Development"** by Jonathan LeBlanc

---

## 18. Summary

**Key Takeaways:**

1. **Multi-Protocol Support**: OAuth 2.0/OIDC for modern apps, SAML 2.0 for enterprise
2. **Stateless Tokens**: JWT access tokens (stateless, scalable)
3. **Token Lifecycle**: Short-lived access tokens (15 min), refresh token rotation
4. **Identity Federation**: Connect with external providers (Google, Microsoft, LDAP)
5. **Multi-Tenancy**: Isolated identity data per organization (sharding)
6. **High Availability**: 99.99% uptime, multi-region deployment
7. **Security**: Token signing, encryption, MFA, token blacklist

**Design Philosophy**: **Security and scalability over simplicity** - multiple protocols, token management, and identity federation prioritize security and enterprise requirements, even if it means more complexity.

