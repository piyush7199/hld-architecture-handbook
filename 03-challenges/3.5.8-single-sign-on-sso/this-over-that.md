# Single Sign-On (SSO) System - Design Decisions (This Over That)

This document provides in-depth analysis of all major architectural decisions made when designing a Single Sign-On (SSO) system like Okta, Auth0, or Microsoft Azure AD.

---

## Table of Contents

1. [OAuth 2.0 vs SAML 2.0](#1-oauth-20-vs-saml-20)
2. [JWT vs Session-Based Authentication](#2-jwt-vs-session-based-authentication)
3. [Stateless vs Stateful Token Management](#3-stateless-vs-stateful-token-management)
4. [Short-Lived vs Long-Lived Access Tokens](#4-short-lived-vs-long-lived-access-tokens)
5. [Token Rotation vs Static Refresh Tokens](#5-token-rotation-vs-static-refresh-tokens)
6. [HttpOnly Cookies vs localStorage for Token Storage](#6-httponly-cookies-vs-localstorage-for-token-storage)
7. [PostgreSQL vs NoSQL for Identity Database](#7-postgresql-vs-nosql-for-identity-database)
8. [Redis vs Memcached for Token Caching](#8-redis-vs-memcached-for-token-caching)
9. [Multi-Tenant Sharding vs Single Database](#9-multi-tenant-sharding-vs-single-database)
10. [bcrypt vs Argon2 for Password Hashing](#10-bcrypt-vs-argon2-for-password-hashing)

---

## 1. OAuth 2.0 vs SAML 2.0

### The Problem

Choose the authentication protocol for SSO - modern OAuth 2.0/OIDC or enterprise SAML 2.0.

### Options Considered

| Feature | OAuth 2.0 / OIDC | SAML 2.0 |
|---------|------------------|----------|
| **Modern Apps** | ✅ Excellent (mobile, web, APIs) | ⚠️ Limited (web-only) |
| **Enterprise** | ⚠️ Growing adoption | ✅ Industry standard |
| **Stateless** | ✅ JWT tokens (stateless) | ❌ XML assertions (stateful) |
| **Performance** | ✅ Fast (JSON, JWT) | ⚠️ Slower (XML parsing) |
| **Mobile Support** | ✅ Native support | ❌ Not suitable for mobile |
| **Fine-Grained Permissions** | ✅ Scopes (read, write, admin) | ⚠️ Attribute-based |
| **Token Format** | ✅ JSON (lightweight) | ❌ XML (verbose) |
| **Active Directory Integration** | ⚠️ Requires OIDC | ✅ Native SAML support |
| **Single Sign-Out** | ⚠️ Limited support | ✅ Full SLO support |
| **Complexity** | ⚠️ Medium (multiple grant types) | ❌ High (XML, complex protocol) |

### Decision Made

**Support Both OAuth 2.0/OIDC and SAML 2.0**

### Rationale

1. **Market Requirements**: Modern apps need OAuth, enterprise needs SAML
2. **OAuth 2.0/OIDC**: Industry standard for modern applications (mobile, web, APIs)
3. **SAML 2.0**: Enterprise standard (Active Directory, legacy systems)
4. **Flexibility**: Support both protocols (applications choose based on needs)

**Why NOT OAuth-Only:**

- **Enterprise Customers**: Require SAML for Active Directory integration
- **Legacy Systems**: Many enterprise apps only support SAML
- **Market Share**: Lose enterprise customers without SAML

**Why NOT SAML-Only:**

- **Mobile Apps**: SAML not suitable for mobile (XML overhead)
- **Modern APIs**: OAuth is standard for REST APIs
- **Performance**: OAuth faster than SAML (JSON vs XML)

### Implementation Details

**OAuth 2.0 / OIDC:**

```
Authorization Code Flow:
  1. User → Application: "Login"
  2. Application → SSO: Redirect to /authorize
  3. SSO → User: Login form
  4. User → SSO: Credentials
  5. SSO → Application: Authorization code
  6. Application → SSO: Exchange code for tokens
  7. SSO → Application: Access token (JWT), refresh token, ID token
```

**SAML 2.0:**

```
SSO Flow:
  1. User → Application: Access resource
  2. Application → SSO: SAML AuthnRequest (XML)
  3. SSO → User: Login form
  4. User → SSO: Credentials
  5. SSO → Application: SAMLResponse (XML assertion)
  6. Application: Validate assertion, create session
```

### Trade-offs Accepted

- **Complexity**: Must implement and maintain both protocols
- **Development Cost**: Two protocol implementations
- **Testing**: More test cases (OAuth + SAML flows)

### When to Reconsider

- If market only requires one protocol (rare)
- If cost constraints prevent dual protocol support

---

## 2. JWT vs Session-Based Authentication

### The Problem

Choose between stateless JWT tokens or stateful session-based authentication.

### Options Considered

| Feature | JWT (Stateless) | Sessions (Stateful) |
|---------|-----------------|---------------------|
| **Server Storage** | ✅ None (stateless) | ❌ Requires Redis/database |
| **Scalability** | ✅ Excellent (horizontal scaling) | ⚠️ Limited (shared session store) |
| **Token Revocation** | ❌ Can't revoke immediately | ✅ Can revoke immediately |
| **Token Size** | ⚠️ Larger (includes claims) | ✅ Smaller (just session ID) |
| **Performance** | ✅ Fast (no database lookup) | ⚠️ Slower (session lookup) |
| **Self-Contained** | ✅ Includes user info | ❌ Requires session lookup |
| **Cross-Service** | ✅ Works across services | ⚠️ Requires shared session store |
| **Complexity** | ⚠️ JWT signing, validation | ✅ Simpler (session ID) |

### Decision Made

**Hybrid Approach: JWT Access Tokens + Stateful Refresh Tokens**

### Rationale

1. **Best of Both**: JWT for access tokens (stateless, scalable), stateful for refresh tokens (revocation)
2. **Scalability**: JWT access tokens scale horizontally (no shared state)
3. **Revocation**: Stateful refresh tokens can be revoked immediately
4. **Performance**: JWT validation is fast (<50ms, no database lookup)
5. **Industry Standard**: Most SSO providers use this approach (Okta, Auth0)

**Why NOT JWT-Only:**

- **Revocation**: Can't revoke JWT immediately (until expiration)
- **Security**: If token stolen, valid until expiration (15 minutes)

**Why NOT Sessions-Only:**

- **Scalability**: Requires shared session store (Redis), doesn't scale as well
- **Performance**: Session lookup adds latency (database/Redis query)
- **Cross-Service**: Requires shared session store across services

### Implementation Details

**Hybrid Approach:**

```
Access Token (JWT):
  - Stateless (no server storage)
  - Short-lived (15 minutes)
  - Contains user info (sub, email, scope)
  - Signed with RS256 (RSA) or HS256 (HMAC)

Refresh Token (Stateful):
  - Stored in database (encrypted)
  - Long-lived (30 days)
  - Can be revoked immediately
  - Used to get new access tokens
```

**Token Validation:**

```
1. Check Redis cache (token → user mapping)
2. If cache hit → return user info (<1ms)
3. If cache miss → validate JWT signature, expiration
4. Store in cache (TTL = token expiration)
```

### Trade-offs Accepted

- **Complexity**: Two token types, different storage strategies
- **Storage**: Refresh tokens require database storage
- **Token Size**: JWT larger than session ID (acceptable for access tokens)

### When to Reconsider

- If immediate revocation is not required (use JWT-only)
- If scalability is not a concern (use sessions-only)

---

## 3. Stateless vs Stateful Token Management

### The Problem

Should token management be stateless (JWT only) or stateful (database storage)?

### Options Considered

| Feature | Stateless (JWT Only) | Stateful (Database) | Hybrid (JWT + Stateful) |
|---------|---------------------|---------------------|-------------------------|
| **Access Token** | ✅ JWT (stateless) | ❌ Session ID (stateful) | ✅ JWT (stateless) |
| **Refresh Token** | ⚠️ JWT (can't revoke) | ✅ Database (can revoke) | ✅ Database (can revoke) |
| **Scalability** | ✅ Excellent | ⚠️ Limited | ✅ Excellent |
| **Revocation** | ❌ Can't revoke | ✅ Can revoke | ✅ Can revoke (refresh token) |
| **Performance** | ✅ Fast (no lookup) | ⚠️ Slower (lookup) | ✅ Fast (JWT) + Revocation (DB) |
| **Complexity** | ✅ Simple | ✅ Simple | ⚠️ More complex |

### Decision Made

**Hybrid: Stateless Access Tokens + Stateful Refresh Tokens**

### Rationale

1. **Best of Both**: Stateless access tokens (scalable), stateful refresh tokens (revocable)
2. **Performance**: JWT validation is fast (<50ms, no database lookup)
3. **Revocation**: Refresh tokens can be revoked immediately (database)
4. **Scalability**: Access tokens scale horizontally (no shared state)
5. **Security**: Short-lived access tokens (15 min), long-lived refresh tokens (30 days) with rotation

**Why NOT Stateless-Only:**

- **Revocation**: Can't revoke tokens immediately (security risk)
- **Token Theft**: If token stolen, valid until expiration

**Why NOT Stateful-Only:**

- **Scalability**: Requires shared session store (doesn't scale as well)
- **Performance**: Session lookup adds latency

### Implementation Details

**Access Token (Stateless JWT):**

```
{
  "sub": "user-123",
  "email": "user@example.com",
  "scope": "read write",
  "exp": 1516242622,
  "iss": "https://sso.example.com"
}
Signed with RS256 (RSA) or HS256 (HMAC)
```

**Refresh Token (Stateful):**

```
Stored in database:
  - token_hash (hashed refresh token)
  - user_id
  - app_id
  - expires_at
  - revoked (boolean)
```

### Trade-offs Accepted

- **Complexity**: Two token types, different storage strategies
- **Storage**: Refresh tokens require database storage
- **Token Rotation**: Must implement refresh token rotation

### When to Reconsider

- If immediate revocation is not required (use stateless-only)
- If scalability is not a concern (use stateful-only)

---

## 4. Short-Lived vs Long-Lived Access Tokens

### The Problem

What should be the lifetime of access tokens - short (15 minutes) or long (24 hours)?

### Options Considered

| Feature | Short-Lived (15 min) | Long-Lived (24 hours) |
|---------|---------------------|----------------------|
| **Security** | ✅ Less damage if stolen | ❌ More damage if stolen |
| **Revocation** | ✅ Expires quickly | ❌ Valid for 24 hours |
| **User Experience** | ⚠️ More refresh requests | ✅ Fewer refresh requests |
| **Token Theft Impact** | ✅ Limited (15 min window) | ❌ Extended (24 hour window) |
| **Refresh Token Usage** | ✅ Forces refresh token usage | ⚠️ Less refresh token usage |
| **Performance** | ⚠️ More refresh requests | ✅ Fewer refresh requests |

### Decision Made

**Short-Lived Access Tokens (15 minutes)**

### Rationale

1. **Security**: Limits damage if token stolen (15-minute window)
2. **Token Theft**: If access token stolen, only valid for 15 minutes
3. **Refresh Token Usage**: Forces refresh token usage (better security)
4. **Industry Standard**: Most SSO providers use 15-minute access tokens (Okta, Auth0)
5. **Revocation**: Token expires quickly (even if can't revoke immediately)

**Why NOT Long-Lived:**

- **Security Risk**: If token stolen, attacker has access for 24 hours
- **Token Theft Impact**: Extended window for malicious use
- **Revocation**: Can't revoke JWT immediately (valid for 24 hours)

### Implementation Details

**Access Token Lifetime:**

```
Access Token: 15 minutes
Refresh Token: 30 days

Flow:
  1. User logs in → Access token (15 min) + Refresh token (30 days)
  2. Access token expires → Use refresh token to get new access token
  3. Refresh token rotation → New refresh token issued, old one invalidated
```

**Token Refresh:**

```
When access token expires:
  1. Application uses refresh token
  2. SSO issues new access token (15 min) + new refresh token (30 days)
  3. Old refresh token invalidated
```

### Trade-offs Accepted

- **User Experience**: More refresh requests (acceptable, automatic)
- **Performance**: More refresh requests (minimal impact, <100ms)

### When to Reconsider

- If user experience is critical (use longer-lived tokens, but less secure)
- If refresh token infrastructure is unavailable (use longer-lived tokens)

---

## 5. Token Rotation vs Static Refresh Tokens

### The Problem

Should refresh tokens be rotated (new token on each use) or static (same token reused)?

### Options Considered

| Feature | Token Rotation | Static Refresh Tokens |
|---------|----------------|----------------------|
| **Token Theft Detection** | ✅ Detects theft (old token used) | ❌ Can't detect theft |
| **Security** | ✅ Higher security | ⚠️ Lower security |
| **Token Theft Impact** | ✅ Limited (old token invalidated) | ❌ Extended (token valid until expiration) |
| **Complexity** | ⚠️ More complex (rotation logic) | ✅ Simpler (no rotation) |
| **Database Updates** | ⚠️ More updates (invalidate old, create new) | ✅ Fewer updates |
| **Industry Standard** | ✅ Recommended (OAuth 2.1) | ⚠️ Older approach |

### Decision Made

**Refresh Token Rotation**

### Rationale

1. **Security**: Detects token theft (if old token used, it's revoked)
2. **Token Theft Detection**: If attacker steals refresh token, old token invalidated on next use
3. **Industry Standard**: OAuth 2.1 recommends token rotation
4. **Best Practice**: Most SSO providers use token rotation (Okta, Auth0)
5. **Limited Damage**: If token stolen, only valid until next use

**Why NOT Static Refresh Tokens:**

- **Token Theft**: If token stolen, valid for 30 days (security risk)
- **No Detection**: Can't detect if token is stolen
- **Extended Impact**: Attacker has access until token expiration

### Implementation Details

**Token Rotation Flow:**

```
1. Client uses refresh_token to get new access_token
2. Server issues new access_token + new refresh_token
3. Old refresh_token invalidated (marked as revoked)
4. Client updates refresh_token

If old refresh_token used:
  - Server detects it's revoked
  - Returns error (token theft detected)
  - All tokens for user revoked (security measure)
```

**Database Updates:**

```
On token refresh:
  1. UPDATE refresh_tokens SET revoked = true WHERE token_id = old_token_id
  2. INSERT INTO refresh_tokens (token_hash, user_id, expires_at) VALUES (...)
```

### Trade-offs Accepted

- **Complexity**: More complex rotation logic
- **Database Updates**: More updates (invalidate old, create new)
- **Error Handling**: Must handle concurrent refresh requests

### When to Reconsider

- If complexity is a concern (use static tokens, but less secure)
- If database performance is critical (use static tokens, fewer updates)

---

## 6. HttpOnly Cookies vs localStorage for Token Storage

### The Problem

Where should tokens be stored on the client - HttpOnly cookies or localStorage?

### Options Considered

| Feature | HttpOnly Cookies | localStorage |
|---------|------------------|--------------|
| **XSS Protection** | ✅ Not accessible to JavaScript | ❌ Accessible to JavaScript |
| **CSRF Protection** | ⚠️ Vulnerable (mitigate with SameSite) | ✅ Not vulnerable |
| **Automatic Sending** | ✅ Sent automatically with requests | ❌ Must manually add to requests |
| **Size Limit** | ⚠️ 4KB limit | ✅ No limit |
| **Cross-Domain** | ⚠️ Domain-specific | ✅ Can be shared |
| **Security** | ✅ More secure (XSS protection) | ❌ Less secure (XSS risk) |
| **Mobile Support** | ⚠️ Limited (web-only) | ✅ Works on mobile |

### Decision Made

**HttpOnly Cookies for Refresh Tokens, Memory/HttpOnly for Access Tokens**

### Rationale

1. **XSS Protection**: HttpOnly cookies not accessible to JavaScript (prevents XSS attacks)
2. **Security**: More secure than localStorage (XSS protection)
3. **Automatic Sending**: Cookies sent automatically with requests (better UX)
4. **Industry Best Practice**: Recommended by OWASP, most SSO providers use cookies
5. **Refresh Tokens**: HttpOnly cookies for refresh tokens (most sensitive)

**Why NOT localStorage-Only:**

- **XSS Risk**: Accessible to JavaScript (XSS attacks can steal tokens)
- **Security**: Less secure than HttpOnly cookies
- **Best Practice**: Not recommended by security experts

**Hybrid Approach:**

- **Refresh Token**: HttpOnly cookie (most secure, can't be accessed by JavaScript)
- **Access Token**: HttpOnly cookie or memory (short-lived, less sensitive)

### Implementation Details

**HttpOnly Cookie:**

```
Set-Cookie: refresh_token=abc123; HttpOnly; Secure; SameSite=Strict; Max-Age=2592000

Properties:
  - HttpOnly: Not accessible to JavaScript (XSS protection)
  - Secure: Only sent over HTTPS
  - SameSite=Strict: CSRF protection
  - Max-Age: 30 days (refresh token lifetime)
```

**CSRF Protection:**

```
SameSite=Strict: Prevents CSRF attacks
CSRF Token: Additional protection for state-changing operations
```

### Trade-offs Accepted

- **CSRF Vulnerability**: HttpOnly cookies vulnerable to CSRF (mitigate with SameSite)
- **Mobile Support**: Limited (web-only, mobile apps use different storage)
- **Size Limit**: 4KB limit (acceptable for tokens)

### When to Reconsider

- If mobile app support is critical (use memory storage for mobile)
- If CSRF is a major concern (use localStorage + CSRF tokens, but XSS risk)

---

## 7. PostgreSQL vs NoSQL for Identity Database

### The Problem

What database should be used for identity storage - PostgreSQL (SQL) or NoSQL (MongoDB, Cassandra)?

### Options Considered

| Feature | PostgreSQL | MongoDB | Cassandra |
|---------|-----------|---------|-----------|
| **ACID Transactions** | ✅ Full ACID | ⚠️ Limited (4.0+) | ❌ Eventual consistency |
| **Multi-Row Transactions** | ✅ Yes | ✅ Yes (4.0+) | ❌ Single partition only |
| **Referential Integrity** | ✅ Foreign keys | ❌ Application-level | ❌ Application-level |
| **Query Flexibility** | ✅ Full SQL, JOINs | ✅ Rich queries | ❌ Limited to partition key |
| **Identity Industry Use** | ✅ Standard | ⚠️ Growing | ❌ Rare |
| **Scaling** | ⚠️ Vertical, sharding needed | ✅ Horizontal | ✅ Horizontal |
| **Maturity** | ✅ Battle-tested | ⚠️ Newer | ✅ Battle-tested |

### Decision Made

**PostgreSQL with Horizontal Sharding**

### Rationale

1. **ACID Requirements**: Identity data requires ACID (user creation, token issuance)
2. **Referential Integrity**: Foreign keys prevent orphaned records
3. **Query Flexibility**: Complex queries (user search, token lookup)
4. **Industry Standard**: Most SSO providers use PostgreSQL (Okta, Auth0)
5. **Maturity**: Battle-tested for identity systems

**Why NOT MongoDB:**

- **Transactions**: Limited transactions (added in 4.0, less battle-tested)
- **Referential Integrity**: Application-level (error-prone)
- **Identity Industry**: Less common in identity systems

**Why NOT Cassandra:**

- **Eventual Consistency**: Unacceptable for identity data (can't have "eventually correct" user)
- **No Transactions**: Can't atomically create user + issue token
- **Query Limitations**: Limited to partition key queries

### Implementation Details

**Sharding Strategy:**

```
Shard by tenant_id:
  shard_id = hash(tenant_id) % 64

Benefits:
  - All tenant data on same shard (fast queries)
  - Even distribution across shards
  - 1,750 QPS / 64 shards = ~27 QPS per shard
```

**Schema:**

```sql
CREATE TABLE users (
    user_id UUID PRIMARY KEY,
    tenant_id UUID NOT NULL,
    email TEXT UNIQUE NOT NULL,
    password_hash TEXT,
    ...
    FOREIGN KEY (tenant_id) REFERENCES tenants(tenant_id)
);

CREATE INDEX idx_tenant_email ON users(tenant_id, email);
```

### Trade-offs Accepted

- **Scaling**: Requires sharding (complexity)
- **Vertical Scaling**: Limited (must shard for horizontal scaling)

### When to Reconsider

- If eventual consistency is acceptable (use Cassandra, but not recommended)
- If document model fits better (use MongoDB, but less common)

---

## 8. Redis vs Memcached for Token Caching

### The Problem

What caching solution should be used for token validation - Redis or Memcached?

### Options Considered

| Feature | Redis | Memcached |
|---------|-------|-----------|
| **Data Structures** | ✅ Rich (strings, hashes, sets, sorted sets) | ⚠️ Simple (key-value only) |
| **Persistence** | ✅ Optional (RDB, AOF) | ❌ No persistence |
| **Replication** | ✅ Built-in | ⚠️ Limited |
| **Pub/Sub** | ✅ Yes | ❌ No |
| **Lua Scripting** | ✅ Yes | ❌ No |
| **Performance** | ✅ Fast | ✅ Fast (similar) |
| **Memory Efficiency** | ⚠️ Less efficient | ✅ More efficient |
| **Complexity** | ⚠️ More features | ✅ Simpler |
| **Industry Use** | ✅ Common for caching | ✅ Common for caching |

### Decision Made

**Redis**

### Rationale

1. **Rich Data Structures**: Hashes for token → user mapping, sets for blacklists
2. **Persistence**: Optional persistence (can recover cache on restart)
3. **Replication**: Built-in replication (high availability)
4. **Pub/Sub**: Can use for token revocation notifications
5. **Industry Standard**: Most SSO providers use Redis (Okta, Auth0)

**Why NOT Memcached:**

- **Limited Features**: Simple key-value only (less flexible)
- **No Persistence**: Cache lost on restart (must rebuild)
- **No Replication**: Limited replication support

### Implementation Details

**Token Cache:**

```
Redis Hash:
  Key: token:eyJ...
  Value: {
    user_id: "user-123",
    email: "user@example.com",
    scope: "read write"
  }
  TTL: 900 seconds (15 minutes, access token lifetime)
```

**Token Blacklist:**

```
Redis Set:
  Key: blacklist:eyJ...
  Value: user_id
  TTL: 900 seconds (access token expiration)
```

**Cache Hit Rate:**

```
95% cache hit rate (most tokens validated multiple times)
Average latency: <5ms (cache hit) vs <50ms (cache miss)
```

### Trade-offs Accepted

- **Memory**: Redis uses more memory than Memcached (acceptable)
- **Complexity**: More features (but useful for token management)

### When to Reconsider

- If memory efficiency is critical (use Memcached, but lose features)
- If simple key-value is sufficient (use Memcached, but less flexible)

---

## 9. Multi-Tenant Sharding vs Single Database

### The Problem

How should multi-tenant data be stored - sharded by tenant or single database?

### Options Considered

| Feature | Sharded by Tenant | Single Database |
|---------|------------------|-----------------|
| **Scalability** | ✅ Horizontal scaling | ❌ Vertical scaling only |
| **Isolation** | ✅ Complete isolation | ⚠️ Row-level security |
| **Performance** | ✅ Fast (all tenant data on same shard) | ⚠️ Slower (larger database) |
| **Complexity** | ⚠️ Sharding logic, routing | ✅ Simpler |
| **Cross-Tenant Queries** | ❌ Difficult (query all shards) | ✅ Easy (single query) |
| **Rebalancing** | ⚠️ Complex | ✅ Not needed |
| **Cost** | ⚠️ Higher (multiple databases) | ✅ Lower (single database) |

### Decision Made

**Sharded by Tenant ID (64 shards)**

### Rationale

1. **Scalability**: Horizontal scaling (64 shards, can add more)
2. **Isolation**: Complete tenant isolation (security, compliance)
3. **Performance**: Fast queries (all tenant data on same shard)
4. **Industry Standard**: Most multi-tenant SaaS use sharding (Okta, Auth0)
5. **Compliance**: Easier to meet data residency requirements (shard by region)

**Why NOT Single Database:**

- **Scalability**: Limited to vertical scaling (expensive)
- **Performance**: Slower queries (larger database)
- **Isolation**: Row-level security (less secure than sharding)

### Implementation Details

**Sharding Strategy:**

```
Shard Calculation:
  shard_id = hash(tenant_id) % 64

Routing:
  - All queries include tenant_id
  - Router calculates shard_id
  - Route query to appropriate shard

Benefits:
  - All tenant data on same shard (fast queries)
  - Even distribution across shards
  - 1,750 QPS / 64 shards = ~27 QPS per shard
```

**Isolation:**

```
Row-Level Security (PostgreSQL):
  CREATE POLICY tenant_isolation ON users
    USING (tenant_id = current_setting('app.tenant_id')::uuid);

All queries automatically filtered by tenant_id
```

### Trade-offs Accepted

- **Complexity**: Sharding logic, routing, cross-shard queries difficult
- **Cost**: Higher infrastructure cost (64 databases)
- **Rebalancing**: Complex if shard distribution becomes uneven

### When to Reconsider

- If scale is small (<1M users, single database sufficient)
- If cost is critical (use single database with row-level security)

---

## 10. bcrypt vs Argon2 for Password Hashing

### The Problem

What password hashing algorithm should be used - bcrypt or Argon2?

### Options Considered

| Feature | bcrypt | Argon2 |
|---------|--------|--------|
| **Industry Standard** | ✅ NIST recommended, widely used | ✅ Winner of PHC (2015) |
| **Memory Hardness** | ❌ CPU-only | ✅ Memory-hard (resistant to ASIC) |
| **Platform Support** | ✅ Universal (built into most languages) | ⚠️ Requires library |
| **Performance** | ✅ Fast (configurable cost factor) | ⚠️ Slower (memory-intensive) |
| **Mobile Optimization** | ✅ Fast on mobile | ⚠️ Memory-intensive (battery drain) |
| **Iterations** | ✅ Configurable (cost factor) | ✅ Configurable (time, memory) |
| **Security** | ✅ Strong (with high cost factor) | ✅ Strong (memory-hard) |
| **Battle-Tested** | ✅ Decades of use | ⚠️ Newer (less battle-tested) |

### Decision Made

**bcrypt with Cost Factor 10**

### Rationale

1. **Industry Standard**: NIST recommended, widely adopted
2. **Platform Support**: Built into most languages (Node.js, Python, Go)
3. **Mobile Performance**: Fast on mobile devices (low battery impact)
4. **Battle-Tested**: Decades of use in production systems
5. **Configurable**: Cost factor 10 = 2^10 = 1,024 iterations (~100ms)

**Why NOT Argon2:**

- **Memory-Intensive**: High memory usage (battery drain on mobile)
- **Library Dependency**: Requires external library (larger app size)
- **Mobile Optimization**: Designed for servers, not mobile devices
- **Battle-Tested**: Newer, less battle-tested than bcrypt

### Implementation Details

**bcrypt Hashing:**

```
Password Hashing:
  password_hash = bcrypt.hash(password, cost_factor=10)
  
Cost Factor 10:
  - 2^10 = 1,024 iterations
  - ~100ms on modern hardware
  - Strong security

Password Verification:
  valid = bcrypt.compare(password, password_hash)
  - Constant time comparison (prevents timing attacks)
```

**Security Properties:**

- **Salt**: Random salt per password (prevents rainbow table attacks)
- **Cost Factor**: Configurable (can increase if needed)
- **Constant Time**: Comparison is constant time (prevents timing attacks)

### Trade-offs Accepted

- **ASIC Vulnerability**: bcrypt is CPU-only (vulnerable to ASIC attacks, but acceptable for mobile)
- **Not Memory-Hard**: Argon2 is more resistant to ASIC, but too memory-intensive for mobile

### When to Reconsider

- If Argon2 becomes standard on mobile platforms (currently not)
- If higher security is required (increase bcrypt cost factor to 12+)

---

## Summary

| Decision | Choice | Key Rationale |
|----------|--------|---------------|
| **Protocol** | OAuth 2.0/OIDC + SAML 2.0 | Market requirements (modern + enterprise) |
| **Token Type** | JWT Access + Stateful Refresh | Scalability (JWT) + Revocation (stateful) |
| **Access Token Lifetime** | 15 minutes | Security (limits damage if stolen) |
| **Refresh Token** | Rotation | Security (detects token theft) |
| **Token Storage** | HttpOnly Cookies | Security (XSS protection) |
| **Database** | PostgreSQL (Sharded) | ACID, referential integrity, industry standard |
| **Cache** | Redis | Rich data structures, persistence, replication |
| **Sharding** | By Tenant ID (64 shards) | Scalability, isolation, performance |
| **Password Hashing** | bcrypt (cost 10) | Industry standard, mobile-optimized, battle-tested |

**Design Philosophy**: **Security and scalability over simplicity** - multiple protocols, token management, and identity federation prioritize security and enterprise requirements, even if it means more complexity.

