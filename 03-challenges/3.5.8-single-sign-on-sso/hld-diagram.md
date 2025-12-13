# Single Sign-On (SSO) System - High-Level Design

## Table of Contents

1. [System Architecture Overview](#system-architecture-overview)
2. [OAuth 2.0 / OIDC Architecture](#oauth-20--oidc-architecture)
3. [SAML 2.0 Architecture](#saml-20-architecture)
4. [Token Management Architecture](#token-management-architecture)
5. [Multi-Tenant Architecture](#multi-tenant-architecture)
6. [Identity Federation Architecture](#identity-federation-architecture)
7. [Session Management Architecture](#session-management-architecture)
8. [Token Validation Architecture](#token-validation-architecture)
9. [Database Sharding Architecture](#database-sharding-architecture)
10. [Multi-Region Deployment](#multi-region-deployment)
11. [Security Architecture](#security-architecture)
12. [Caching Architecture](#caching-architecture)

---

## System Architecture Overview

**Flow Explanation:**

This diagram shows the high-level architecture of the SSO system, including authentication APIs, token validation, identity database, and external identity providers.

**Key Components:**

1. **Applications**: Web apps, mobile apps, APIs that use SSO
2. **API Gateway**: Rate limiting, routing, TLS termination
3. **Auth API**: Handles login, token issuance
4. **Token API**: Validates tokens, highest QPS (50k QPS)
5. **User API**: User profile management
6. **Identity Database**: PostgreSQL, sharded by tenant_id
7. **Token Cache**: Redis for fast token validation
8. **External Providers**: Google, Microsoft, LDAP, Active Directory

**Benefits:**

- **Scalability**: Stateless services (horizontal scaling)
- **Performance**: Token caching (95% hit rate)
- **Multi-Protocol**: OAuth, SAML, OIDC support
- **Federation**: Connect with external providers

**Trade-offs:**

- Complexity: Multiple protocols, token management
- Dependency: Relies on external providers for federation

```mermaid
graph TB
    subgraph "Applications"
        Web[Web Apps]
        Mobile[Mobile Apps]
        API[APIs]
    end
    
    subgraph "SSO System"
        Gateway[API Gateway<br/>Rate Limiting, Routing]
        AuthAPI[Auth API<br/>Login, Token Issuance]
        TokenAPI[Token API<br/>Token Validation<br/>50k QPS]
        UserAPI[User API<br/>Profile Management]
        
        Cache[Token Cache<br/>Redis Cluster]
        DB[(Identity Database<br/>PostgreSQL<br/>Sharded by Tenant)]
    end
    
    subgraph "External Providers"
        Google[Google OAuth]
        Microsoft[Microsoft Identity]
        LDAP[LDAP / Active Directory]
    end
    
    Web --> Gateway
    Mobile --> Gateway
    API --> Gateway
    
    Gateway --> AuthAPI
    Gateway --> TokenAPI
    Gateway --> UserAPI
    
    AuthAPI --> DB
    AuthAPI --> Cache
    TokenAPI --> Cache
    TokenAPI --> DB
    UserAPI --> DB
    
    Cache --> DB
    
    AuthAPI -.->|Federation| Google
    AuthAPI -.->|Federation| Microsoft
    AuthAPI -.->|Federation| LDAP
    
    style Gateway fill:#e1f5ff
    style TokenAPI fill:#fff4e1
    style Cache fill:#ffe1f5
    style DB fill:#e1ffe1
```

---

## OAuth 2.0 / OIDC Architecture

**Flow Explanation:**

This diagram shows the OAuth 2.0 / OIDC architecture, including authorization server, resource server, and token flow.

**Key Components:**

1. **Authorization Server**: Issues access tokens, refresh tokens, ID tokens
2. **Resource Server**: Validates tokens, serves protected resources
3. **Client Application**: Requests authorization, uses tokens
4. **User**: Grants authorization, authenticates

**Flow:**

1. User requests access → Client redirects to Authorization Server
2. User authenticates → Authorization Server issues authorization code
3. Client exchanges code → Authorization Server issues tokens
4. Client uses access token → Resource Server validates token
5. Resource Server grants access → Returns protected resource

**Benefits:**

- **Stateless**: JWT tokens (no server-side session)
- **Scalable**: Horizontal scaling (no shared state)
- **Secure**: Industry-standard protocol

**Trade-offs:**

- Token revocation: Can't revoke JWT immediately (until expiration)
- Complexity: Multiple grant types, scopes

```mermaid
graph TB
    User[User] --> Client[Client Application]
    Client -->|1. Redirect| AuthServer[Authorization Server<br/>SSO Provider]
    AuthServer -->|2. Login Form| User
    User -->|3. Credentials| AuthServer
    AuthServer -->|4. Authorization Code| Client
    Client -->|5. Exchange Code| AuthServer
    AuthServer -->|6. Access Token + Refresh Token| Client
    Client -->|7. Access Token| ResourceServer[Resource Server<br/>Application Backend]
    ResourceServer -->|8. Validate Token| AuthServer
    AuthServer -->|9. User Info| ResourceServer
    ResourceServer -->|10. Protected Resource| Client
    
    style AuthServer fill:#e1f5ff
    style ResourceServer fill:#fff4e1
    style Client fill:#e1ffe1
```

---

## SAML 2.0 Architecture

**Flow Explanation:**

This diagram shows the SAML 2.0 architecture for enterprise SSO, including identity provider (IdP) and service provider (SP).

**Key Components:**

1. **Identity Provider (IdP)**: SSO provider, authenticates users, issues SAML assertions
2. **Service Provider (SP)**: Application, consumes SAML assertions
3. **User**: Accesses application, redirected to IdP for authentication

**Flow:**

1. User accesses application → SP redirects to IdP
2. User authenticates → IdP generates SAML assertion (XML)
3. IdP redirects to SP → SP receives SAML assertion
4. SP validates assertion → Creates session
5. User accesses application → Session-based access

**Benefits:**

- **Enterprise Standard**: Active Directory integration
- **Single Sign-Out**: SLO support
- **Attribute-Based**: Rich user attributes in assertion

**Trade-offs:**

- **Complexity**: XML parsing, complex protocol
- **Performance**: Slower than OAuth (XML overhead)

```mermaid
graph TB
    User[User] -->|1. Access App| SP[Service Provider<br/>Application]
    SP -->|2. Redirect| IdP[Identity Provider<br/>SSO System]
    IdP -->|3. Login Form| User
    User -->|4. Credentials| IdP
    IdP -->|5. Generate SAML Assertion| IdP
    IdP -->|6. POST SAMLResponse| SP
    SP -->|7. Validate Assertion| SP
    SP -->|8. Create Session| SP
    SP -->|9. Access Granted| User
    
    style IdP fill:#e1f5ff
    style SP fill:#fff4e1
```

---

## Token Management Architecture

**Flow Explanation:**

This diagram shows the token management architecture, including access tokens, refresh tokens, ID tokens, and token lifecycle.

**Token Types:**

1. **Access Token (JWT)**: Short-lived (15 minutes), contains user info and permissions
2. **Refresh Token**: Long-lived (30 days), used to get new access tokens
3. **ID Token (JWT)**: Contains user identity (OIDC), short-lived (15 minutes)

**Token Storage:**

- **Access Token**: Client-side (HttpOnly cookie or memory)
- **Refresh Token**: HttpOnly cookie (more secure) or encrypted database
- **Token Blacklist**: Redis for revoked tokens (TTL = token expiration)

**Token Rotation:**

- Client uses refresh_token → Server issues new access_token + new refresh_token
- Old refresh_token invalidated → Prevents token theft

**Benefits:**

- **Security**: Short-lived access tokens (limits damage if stolen)
- **User Experience**: Refresh tokens (fewer logins)
- **Revocation**: Token blacklist (immediate revocation)

**Trade-offs:**

- **Complexity**: Token rotation, blacklist management
- **Storage**: Refresh tokens require database storage

```mermaid
graph TB
    subgraph "Token Issuance"
        Login[User Login] --> Auth[Auth API]
        Auth --> AccessToken[Access Token<br/>JWT, 15 min]
        Auth --> RefreshToken[Refresh Token<br/>30 days]
        Auth --> IDToken[ID Token<br/>JWT, 15 min]
    end
    
    subgraph "Token Storage"
        AccessToken --> Client[Client Storage<br/>HttpOnly Cookie / Memory]
        RefreshToken --> DB[(Database<br/>Encrypted)]
        IDToken --> Client
    end
    
    subgraph "Token Validation"
        Request[API Request] --> Validate[Token Validation API]
        Validate --> Cache[Redis Cache<br/>Token → User Mapping]
        Cache -->|Cache Hit| UserInfo[User Info]
        Cache -->|Cache Miss| JWTValidate[JWT Validation<br/>Signature, Expiration]
        JWTValidate --> Blacklist[Token Blacklist<br/>Redis]
        Blacklist --> UserInfo
    end
    
    subgraph "Token Refresh"
        Expired[Access Token Expired] --> Refresh[Refresh Token API]
        Refresh --> NewAccess[New Access Token]
        Refresh --> NewRefresh[New Refresh Token]
        Refresh --> Invalidate[Invalidate Old Refresh Token]
    end
    
    style AccessToken fill:#e1f5ff
    style RefreshToken fill:#fff4e1
    style Cache fill:#ffe1f5
    style DB fill:#e1ffe1
```

---

## Multi-Tenant Architecture

**Flow Explanation:**

This diagram shows the multi-tenant architecture with tenant isolation, sharding, and data segregation.

**Sharding Strategy:**

- Shard by tenant_id: hash(tenant_id) % 64
- All tenant data on same shard (fast queries)
- Even distribution across shards

**Isolation:**

- Row-level security (PostgreSQL)
- Tenant context in all queries
- Separate client_id per tenant

**Benefits:**

- **Isolation**: Tenant data completely isolated
- **Performance**: Fast queries (all tenant data on same shard)
- **Scalability**: Horizontal scaling (64 shards)

**Trade-offs:**

- **Complexity**: Sharding logic, tenant context management
- **Cross-Tenant Queries**: Difficult (requires querying all shards)

```mermaid
graph TB
    subgraph "Applications"
        App1[App 1<br/>Tenant A]
        App2[App 2<br/>Tenant B]
        App3[App 3<br/>Tenant C]
    end
    
    subgraph "SSO System"
        Gateway[API Gateway]
        Auth[Auth API]
        Router[Tenant Router<br/>hash tenant_id % 64]
    end
    
    subgraph "Database Shards"
        Shard1[Shard 1<br/>Tenants: A, D, G...]
        Shard2[Shard 2<br/>Tenants: B, E, H...]
        Shard3[Shard 3<br/>Tenants: C, F, I...]
        ShardN[Shard 64<br/>...]
    end
    
    App1 --> Gateway
    App2 --> Gateway
    App3 --> Gateway
    
    Gateway --> Auth
    Auth --> Router
    
    Router -->|tenant_id % 64 = 1| Shard1
    Router -->|tenant_id % 64 = 2| Shard2
    Router -->|tenant_id % 64 = 3| Shard3
    Router -->|tenant_id % 64 = 64| ShardN
    
    style Router fill:#e1f5ff
    style Shard1 fill:#fff4e1
    style Shard2 fill:#fff4e1
    style Shard3 fill:#fff4e1
```

---

## Identity Federation Architecture

**Flow Explanation:**

This diagram shows the identity federation architecture, connecting with external identity providers (Google, Microsoft, LDAP).

**Federation Types:**

1. **Social Login**: Google, Facebook, Microsoft, GitHub, Apple
2. **Enterprise**: LDAP, Active Directory, Azure AD
3. **Other SSO Providers**: Okta, Auth0 (federation)

**Flow:**

1. User selects "Login with Google" → SSO redirects to Google
2. User authenticates with Google → Google returns authorization code
3. SSO exchanges code → Google returns user info
4. SSO creates/links local account → Issues access token
5. User accesses application → Uses SSO access token

**Benefits:**

- **User Experience**: No password creation (uses existing account)
- **Security**: Leverages provider's security (Google, Microsoft)
- **Flexibility**: Multiple identity sources

**Trade-offs:**

- **Dependency**: Relies on external providers (availability)
- **Complexity**: Multiple provider integrations

```mermaid
graph TB
    User[User] --> SSO[SSO System]
    SSO -->|1. Redirect| Google[Google OAuth]
    SSO -->|1. Redirect| Microsoft[Microsoft Identity]
    SSO -->|1. Redirect| LDAP[LDAP / Active Directory]
    
    Google -->|2. User Authenticates| Google
    Microsoft -->|2. User Authenticates| Microsoft
    LDAP -->|2. User Authenticates| LDAP
    
    Google -->|3. Authorization Code| SSO
    Microsoft -->|3. Authorization Code| SSO
    LDAP -->|3. User Credentials| SSO
    
    SSO -->|4. Exchange Code| Google
    SSO -->|4. Validate Credentials| LDAP
    
    Google -->|5. User Info| SSO
    Microsoft -->|5. User Info| SSO
    LDAP -->|5. User Info| SSO
    
    SSO -->|6. Create/Link Account| DB[(Identity Database)]
    SSO -->|7. Issue Access Token| User
    User -->|8. Access Application| App[Application]
    
    style SSO fill:#e1f5ff
    style Google fill:#fff4e1
    style Microsoft fill:#fff4e1
    style LDAP fill:#fff4e1
    style DB fill:#e1ffe1
```

---

## Session Management Architecture

**Flow Explanation:**

This diagram shows the session management architecture, including stateless JWT tokens and stateful refresh tokens.

**Session Types:**

1. **Stateless (JWT)**: Access token contains all user info, no server-side storage
2. **Stateful (Refresh Token)**: Stored in database, can be revoked immediately

**Hybrid Approach:**

- Access token: JWT (stateless, 15 minutes)
- Refresh token: Stateful (Redis/database, 30 days)
- Benefits: Fast validation (JWT), immediate revocation (refresh token)

**Benefits:**

- **Scalability**: Stateless access tokens (horizontal scaling)
- **Revocation**: Stateful refresh tokens (immediate revocation)
- **Performance**: JWT validation (<50ms)

**Trade-offs:**

- **Complexity**: Two token types, different storage strategies
- **Storage**: Refresh tokens require database storage

```mermaid
graph TB
    subgraph "Stateless (Access Token)"
        JWT[Access Token<br/>JWT]
        JWT --> Claims[User Claims<br/>sub, email, scope]
        JWT --> Signature[Signature<br/>RS256 / HS256]
        JWT --> Expiration[Expiration<br/>15 minutes]
    end
    
    subgraph "Stateful (Refresh Token)"
        Refresh[Refresh Token<br/>Random String]
        Refresh --> DB[(Database<br/>Encrypted)]
        Refresh --> Expires[Expires At<br/>30 days]
        Refresh --> Revoked[Revoked Flag]
    end
    
    subgraph "Session Flow"
        Login[User Login] --> Issue[Issue Tokens]
        Issue --> JWT
        Issue --> Refresh
        
        Request[API Request] --> ValidateJWT[Validate JWT<br/>Signature, Expiration]
        ValidateJWT -->|Valid| Allow[Allow Access]
        ValidateJWT -->|Expired| UseRefresh[Use Refresh Token]
        UseRefresh --> ValidateRefresh[Validate Refresh Token<br/>Check Database]
        ValidateRefresh -->|Valid| NewJWT[Issue New JWT]
        ValidateRefresh -->|Revoked| Deny[Deny Access]
    end
    
    style JWT fill:#e1f5ff
    style Refresh fill:#fff4e1
    style DB fill:#ffe1f5
```

---

## Token Validation Architecture

**Flow Explanation:**

This diagram shows the token validation architecture, including JWT validation, caching, and blacklist checking.

**Validation Flow:**

1. **Cache Lookup**: Check Redis cache (token → user mapping)
2. **Cache Hit**: Return user info (<1ms)
3. **Cache Miss**: Validate JWT signature, expiration, claims
4. **Blacklist Check**: Verify token not revoked
5. **Cache Store**: Store token → user mapping (TTL = token expiration)

**Performance:**

- Cache hit rate: 95% (most tokens validated multiple times)
- Average latency: <5ms (cache hit) vs <50ms (cache miss)
- QPS: 50k QPS (peak)

**Benefits:**

- **Fast**: Redis cache (95% hit rate)
- **Scalable**: Horizontal scaling (Redis cluster)
- **Secure**: JWT signature validation, blacklist checking

**Trade-offs:**

- **Memory**: Redis cache requires memory (token → user mappings)
- **Consistency**: Cache may be stale (acceptable for token validation)

```mermaid
graph TB
    Request[API Request<br/>Authorization: Bearer token] --> Validate[Token Validation API]
    
    Validate --> Cache{Redis Cache<br/>Token → User}
    
    Cache -->|Cache Hit<br/>95%| UserInfo[Return User Info<br/><1ms]
    Cache -->|Cache Miss<br/>5%| JWTValidate[JWT Validation]
    
    JWTValidate --> Signature[Verify Signature<br/>RS256 / HS256]
    JWTValidate --> Expiration[Check Expiration<br/>exp claim]
    JWTValidate --> Claims[Validate Claims<br/>iss, aud, sub]
    
    Signature --> Blacklist{Token Blacklist<br/>Redis}
    Expiration --> Blacklist
    Claims --> Blacklist
    
    Blacklist -->|Not Revoked| StoreCache[Store in Cache<br/>TTL = token expiration]
    Blacklist -->|Revoked| Deny[Deny Access<br/>401 Unauthorized]
    
    StoreCache --> UserInfo
    
    style Cache fill:#e1f5ff
    style JWTValidate fill:#fff4e1
    style Blacklist fill:#ffe1f5
    style UserInfo fill:#e1ffe1
```

---

## Database Sharding Architecture

**Flow Explanation:**

This diagram shows the database sharding architecture, sharding by tenant_id across 64 shards.

**Sharding Strategy:**

- Shard calculation: hash(tenant_id) % 64
- All tenant data on same shard (users, applications, tokens)
- Even distribution across shards

**Benefits:**

- **Performance**: Fast queries (all tenant data on same shard)
- **Scalability**: Horizontal scaling (64 shards)
- **Isolation**: Tenant data completely isolated

**Trade-offs:**

- **Complexity**: Sharding logic, cross-shard queries difficult
- **Rebalancing**: Difficult if shard distribution becomes uneven

```mermaid
graph TB
    subgraph "Applications"
        App1[App 1<br/>Tenant A]
        App2[App 2<br/>Tenant B]
        App3[App 3<br/>Tenant C]
    end
    
    subgraph "SSO System"
        API[Auth API]
        Router[Shard Router<br/>hash tenant_id % 64]
    end
    
    subgraph "Database Shards (64 Total)"
        Shard1[Shard 1<br/>Tenants: A, D, G...<br/>~1.5M users]
        Shard2[Shard 2<br/>Tenants: B, E, H...<br/>~1.5M users]
        Shard3[Shard 3<br/>Tenants: C, F, I...<br/>~1.5M users]
        ShardN[Shard 64<br/>...<br/>~1.5M users]
    end
    
    App1 --> API
    App2 --> API
    App3 --> API
    
    API --> Router
    
    Router -->|Shard 1| Shard1
    Router -->|Shard 2| Shard2
    Router -->|Shard 3| Shard3
    Router -->|Shard 64| ShardN
    
    style Router fill:#e1f5ff
    style Shard1 fill:#fff4e1
    style Shard2 fill:#fff4e1
    style Shard3 fill:#fff4e1
```

---

## Multi-Region Deployment

**Flow Explanation:**

This diagram shows the multi-region deployment architecture for high availability and low latency.

**Regions:**

- **US-East**: Primary region (70% traffic)
- **EU-West**: Secondary region (20% traffic)
- **AP-South**: Tertiary region (10% traffic)

**Components:**

- **Load Balancer**: Global load balancer (route to nearest region)
- **API Services**: Auto-scaling (2-100 instances per region)
- **Database**: PostgreSQL (primary + read replicas per region)
- **Cache**: Redis Cluster (multi-region replication)

**Benefits:**

- **High Availability**: 99.99% uptime (any 2 regions up)
- **Low Latency**: Requests routed to nearest region
- **Disaster Recovery**: Failover to other regions

**Trade-offs:**

- **Complexity**: Multi-region replication, data consistency
- **Cost**: Higher infrastructure cost (3 regions)

```mermaid
graph TB
    Users[Global Users] --> LB[Global Load Balancer<br/>Route to Nearest Region]
    
    LB --> US[US-East Region<br/>70% Traffic]
    LB --> EU[EU-West Region<br/>20% Traffic]
    LB --> AP[AP-South Region<br/>10% Traffic]
    
    subgraph "US-East"
        US_API[Auth API<br/>Auto-scaling]
        US_DB[(Primary DB)]
        US_Cache[(Redis Cluster)]
    end
    
    subgraph "EU-West"
        EU_API[Auth API<br/>Auto-scaling]
        EU_DB[(Read Replica)]
        EU_Cache[(Redis Cluster)]
    end
    
    subgraph "AP-South"
        AP_API[Auth API<br/>Auto-scaling]
        AP_DB[(Read Replica)]
        AP_Cache[(Redis Cluster)]
    end
    
    US --> US_API
    US_API --> US_DB
    US_API --> US_Cache
    
    EU --> EU_API
    EU_API --> EU_DB
    EU_API --> EU_Cache
    
    AP --> AP_API
    AP_API --> AP_DB
    AP_API --> AP_Cache
    
    US_DB -.->|Async Replication| EU_DB
    US_DB -.->|Async Replication| AP_DB
    
    style LB fill:#e1f5ff
    style US_DB fill:#fff4e1
    style EU_DB fill:#e1ffe1
    style AP_DB fill:#e1ffe1
```

---

## Security Architecture

**Flow Explanation:**

This diagram shows the multi-layer security architecture, including encryption, token signing, and secure storage.

**Security Layers:**

1. **Encryption in Transit**: TLS 1.3 for all network communication
2. **Encryption at Rest**: Database encryption (AES-256-GCM)
3. **Token Signing**: JWT signed with RS256 (RSA) or HS256 (HMAC)
4. **Password Hashing**: bcrypt (cost factor 10, ~100ms)
5. **Token Storage**: HttpOnly cookies (XSS protection)
6. **Rate Limiting**: Prevent brute force attacks

**Benefits:**

- **Defense in Depth**: Multiple security layers
- **Industry Standards**: TLS, bcrypt, JWT signing
- **Compliance**: SOC 2, GDPR, HIPAA

**Security Properties:**

- Passwords never stored in plaintext
- Tokens signed and encrypted
- Secure cookie flags (HttpOnly, Secure, SameSite)

```mermaid
graph TB
    subgraph "Network Security"
        TLS[TLS 1.3<br/>Encryption in Transit]
        RateLimit[Rate Limiting<br/>Prevent Brute Force]
    end
    
    subgraph "Application Security"
        Auth[Authentication<br/>Username + Password + MFA]
        TokenSign[Token Signing<br/>RS256 / HS256]
        PasswordHash[Password Hashing<br/>bcrypt, cost 10]
    end
    
    subgraph "Storage Security"
        DBEncrypt[Database Encryption<br/>AES-256-GCM at Rest]
        CookieSecure[Secure Cookies<br/>HttpOnly, Secure, SameSite]
        TokenEncrypt[Token Encryption<br/>Refresh Tokens Encrypted]
    end
    
    subgraph "Access Control"
        RBAC[Role-Based Access Control<br/>RBAC]
        Scopes[OAuth Scopes<br/>Fine-Grained Permissions]
        Blacklist[Token Blacklist<br/>Revoked Tokens]
    end
    
    TLS --> Auth
    RateLimit --> Auth
    Auth --> PasswordHash
    Auth --> TokenSign
    TokenSign --> CookieSecure
    PasswordHash --> DBEncrypt
    TokenEncrypt --> DBEncrypt
    TokenSign --> RBAC
    RBAC --> Scopes
    Scopes --> Blacklist
    
    style TLS fill:#ffe1f5
    style TokenSign fill:#fff4e1
    style DBEncrypt fill:#e1ffe1
    style Blacklist fill:#e1f5ff
```

---

## Caching Architecture

**Flow Explanation:**

This diagram shows the caching architecture for token validation, authorization codes, and session data.

**Cache Layers:**

1. **Token Cache**: Access token → user mapping (Redis, 95% hit rate)
2. **Authorization Code Cache**: Code → user mapping (Redis, 10-minute TTL)
3. **Token Blacklist**: Revoked tokens (Redis, TTL = token expiration)
4. **User Profile Cache**: User profile data (Redis, 1-hour TTL)

**Benefits:**

- **Performance**: Fast token validation (<5ms cache hit)
- **Scalability**: Redis cluster (horizontal scaling)
- **Reduced Load**: 95% cache hit rate (reduces database load)

**Trade-offs:**

- **Memory**: Redis cache requires memory (token → user mappings)
- **Consistency**: Cache may be stale (acceptable for token validation)

```mermaid
graph TB
    subgraph "Cache Layers"
        TokenCache[Token Cache<br/>Token → User Mapping<br/>95% Hit Rate]
        CodeCache[Authorization Code Cache<br/>Code → User Mapping<br/>10-min TTL]
        BlacklistCache[Token Blacklist<br/>Revoked Tokens<br/>TTL = Expiration]
        ProfileCache[User Profile Cache<br/>User → Profile<br/>1-hour TTL]
    end
    
    subgraph "Redis Cluster"
        Redis1[Redis Node 1]
        Redis2[Redis Node 2]
        Redis3[Redis Node 3]
        RedisN[Redis Node N]
    end
    
    subgraph "Cache Operations"
        Validate[Token Validation] --> TokenCache
        AuthCode[Authorization Code] --> CodeCache
        Revoke[Token Revocation] --> BlacklistCache
        Profile[User Profile] --> ProfileCache
    end
    
    TokenCache --> Redis1
    CodeCache --> Redis2
    BlacklistCache --> Redis3
    ProfileCache --> RedisN
    
    Redis1 -.->|Replication| Redis2
    Redis2 -.->|Replication| Redis3
    Redis3 -.->|Replication| RedisN
    
    style TokenCache fill:#e1f5ff
    style Redis1 fill:#fff4e1
    style Redis2 fill:#fff4e1
    style Redis3 fill:#fff4e1
```

