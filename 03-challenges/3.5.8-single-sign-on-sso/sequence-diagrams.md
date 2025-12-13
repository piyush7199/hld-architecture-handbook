# Single Sign-On (SSO) System - Sequence Diagrams

## Table of Contents

1. [OAuth 2.0 Authorization Code Flow](#oauth-20-authorization-code-flow)
2. [SAML 2.0 SSO Flow](#saml-20-sso-flow)
3. [Token Validation Flow](#token-validation-flow)
4. [Token Refresh Flow](#token-refresh-flow)
5. [Social Login Flow (Google OAuth)](#social-login-flow-google-oauth)
6. [Multi-Factor Authentication (MFA) Flow](#multi-factor-authentication-mfa-flow)
7. [Single Sign-Out (SSO Logout) Flow](#single-sign-out-sso-logout-flow)
8. [Identity Federation Flow (LDAP)](#identity-federation-flow-ldap)

---

## OAuth 2.0 Authorization Code Flow

**Flow:**

Shows the complete OAuth 2.0 Authorization Code Flow, from user login request to accessing protected resources with access token.

**Steps:**

1. **User Requests Access** (0ms): User clicks "Login" on application
2. **Application Redirects** (10ms): Application redirects to SSO /authorize endpoint
3. **SSO Shows Login Form** (50ms): SSO displays login form to user
4. **User Enters Credentials** (2000ms): User enters username + password (user time)
5. **SSO Validates Credentials** (2100ms): SSO validates password hash, checks MFA
6. **SSO Generates Authorization Code** (2150ms): SSO generates random code, stores in Redis (10-min TTL)
7. **SSO Redirects with Code** (2200ms): SSO redirects user to application with authorization_code
8. **Application Exchanges Code** (2250ms): Application sends code + client_secret to SSO /token endpoint
9. **SSO Issues Tokens** (2300ms): SSO validates code, issues access_token (JWT), refresh_token, id_token
10. **Application Uses Access Token** (2350ms): Application uses access_token to access protected resources
11. **Resource Validates Token** (2400ms): Resource server validates JWT signature, expiration
12. **Resource Returns Data** (2450ms): Resource server returns protected data to application

**Performance:**

- Total latency: ~2.5 seconds (mostly user time entering credentials)
- Token issuance: <100ms
- Token validation: <50ms (JWT signature check)

**Security:**

- Authorization code: 10-minute TTL (prevents replay attacks)
- Access token: 15-minute lifetime (limits damage if stolen)
- Refresh token: 30-day lifetime (with rotation)

```mermaid
sequenceDiagram
    participant User
    participant App as Application
    participant SSO as SSO Provider
    participant Redis as Redis Cache
    participant Resource as Resource Server
    
    User->>App: Click "Login"
    Note over App: t=0ms
    App->>SSO: Redirect to /authorize<br/>?client_id=...&redirect_uri=...
    Note over SSO: t=10ms
    SSO->>User: Show Login Form
    Note over User: t=50ms
    User->>SSO: Enter Username + Password
    Note over User: t=2000ms (user time)
    SSO->>SSO: Validate Password Hash<br/>bcrypt.compare()
    Note over SSO: t=2100ms
    SSO->>SSO: Generate Authorization Code<br/>random 32 bytes
    Note over SSO: t=2150ms
    SSO->>Redis: Store Code → User Mapping<br/>SET code:abc123 {user_id} EX 600
    Note over Redis: t=2160ms
    SSO->>App: Redirect with Code<br/>?code=abc123&state=...
    Note over App: t=2200ms
    App->>SSO: POST /token<br/>{code: "abc123", client_secret: "..."}
    Note over SSO: t=2250ms
    SSO->>Redis: GET code:abc123
    Redis->>SSO: {user_id: "user-123"}
    SSO->>SSO: Issue Tokens<br/>Access Token (JWT), Refresh Token, ID Token
    Note over SSO: t=2300ms
    SSO->>App: {access_token: "eyJ...", refresh_token: "...", id_token: "eyJ..."}
    App->>Resource: GET /api/data<br/>Authorization: Bearer eyJ...
    Note over Resource: t=2350ms
    Resource->>SSO: Validate Token<br/>Check signature, expiration
    Note over SSO: t=2400ms
    SSO->>Resource: User Info<br/>{user_id: "user-123", scope: "read"}
    Resource->>App: Protected Data
    Note over Resource: t=2450ms
    App->>User: Display Data
```

---

## SAML 2.0 SSO Flow

**Flow:**

Shows the SAML 2.0 SSO flow for enterprise authentication, including SAML request generation, assertion creation, and validation.

**Steps:**

1. **User Accesses Application** (0ms): User tries to access protected resource
2. **Application Generates SAML Request** (10ms): Application generates SAML AuthnRequest (XML)
3. **Application Redirects to SSO** (20ms): Application redirects user to SSO /saml/sso endpoint
4. **SSO Parses SAML Request** (50ms): SSO parses XML, extracts SP entity ID, ACS URL
5. **SSO Shows Login Form** (100ms): SSO displays login form (if user not authenticated)
6. **User Enters Credentials** (2000ms): User enters username + password (user time)
7. **SSO Validates Credentials** (2100ms): SSO validates password, checks MFA
8. **SSO Generates SAML Assertion** (2150ms): SSO creates SAML assertion (XML) with user attributes
9. **SSO Signs Assertion** (2200ms): SSO signs SAML assertion with private key
10. **SSO POSTs to Application** (2250ms): SSO POSTs SAMLResponse to application ACS URL
11. **Application Validates Assertion** (2300ms): Application validates signature, checks expiration
12. **Application Creates Session** (2350ms): Application creates user session
13. **User Accesses Resource** (2400ms): User can now access protected resources

**Performance:**

- Total latency: ~2.4 seconds (mostly user time)
- SAML assertion generation: <100ms
- SAML validation: <50ms

**Security:**

- SAML assertion signed with RSA private key
- Assertion valid for 5 minutes
- NotOnOrAfter timestamp prevents replay

```mermaid
sequenceDiagram
    participant User
    participant SP as Service Provider<br/>Application
    participant SSO as Identity Provider<br/>SSO System
    participant DB as Identity Database
    
    User->>SP: Access Protected Resource
    Note over SP: t=0ms
    SP->>SP: Generate SAML AuthnRequest<br/>(XML)
    Note over SP: t=10ms
    SP->>SSO: Redirect to /saml/sso<br/>?SAMLRequest=... (base64 encoded XML)
    Note over SSO: t=20ms
    SSO->>SSO: Decode & Parse SAMLRequest<br/>Extract SP entity ID, ACS URL
    Note over SSO: t=50ms
    SSO->>User: Show Login Form
    Note over User: t=100ms
    User->>SSO: Enter Username + Password
    Note over User: t=2000ms (user time)
    SSO->>DB: Validate Credentials<br/>SELECT * FROM users WHERE email = ...
    Note over DB: t=2100ms
    DB->>SSO: User Record
    SSO->>SSO: Validate Password Hash<br/>bcrypt.compare()
    SSO->>SSO: Generate SAML Assertion<br/>(XML with user attributes)
    Note over SSO: t=2150ms
    SSO->>SSO: Sign Assertion<br/>RSA-SHA256(private_key, assertion)
    Note over SSO: t=2200ms
    SSO->>SP: POST /saml/acs<br/>{SAMLResponse: "...", RelayState: "..."}
    Note over SP: t=2250ms
    SP->>SP: Validate SAML Assertion<br/>Verify signature, check expiration
    Note over SP: t=2300ms
    SP->>SP: Create User Session<br/>Store session in Redis
    Note over SP: t=2350ms
    SP->>User: Access Granted<br/>Redirect to protected resource
    Note over SP: t=2400ms
```

---

## Token Validation Flow

**Flow:**

Shows the token validation flow, including cache lookup, JWT validation, and blacklist checking.

**Steps:**

1. **Application Sends Request** (0ms): Application sends API request with access token
2. **Resource Server Receives Request** (5ms): Resource server extracts access token from Authorization header
3. **Check Redis Cache** (10ms): Resource server checks Redis cache for token → user mapping
4. **Cache Hit (95% of time)** (15ms): If cache hit, return user info immediately
5. **Cache Miss (5% of time)** (20ms): If cache miss, validate JWT signature
6. **Verify JWT Signature** (25ms): Verify RS256 signature using SSO public key
7. **Check Expiration** (30ms): Check exp claim (token not expired)
8. **Validate Claims** (35ms): Validate iss (issuer), aud (audience), sub (subject)
9. **Check Token Blacklist** (40ms): Verify token not in blacklist (Redis)
10. **Store in Cache** (45ms): Store token → user mapping in Redis (TTL = token expiration)
11. **Return User Info** (50ms): Return user info to resource server
12. **Resource Server Grants Access** (55ms): Resource server grants access based on user info

**Performance:**

- Cache hit: <5ms (95% of requests)
- Cache miss: <50ms (5% of requests)
- Average: <10ms

**Key Points:**

- 95% cache hit rate (most tokens validated multiple times)
- JWT validation only on cache miss
- Token blacklist for immediate revocation

```mermaid
sequenceDiagram
    participant App as Application
    participant Resource as Resource Server
    participant Cache as Redis Cache
    participant SSO as SSO Provider
    participant Blacklist as Token Blacklist<br/>Redis
    
    App->>Resource: GET /api/data<br/>Authorization: Bearer eyJ...
    Note over Resource: t=0ms
    Resource->>Resource: Extract Access Token<br/>from Authorization header
    Note over Resource: t=5ms
    Resource->>Cache: GET token:eyJ...
    Note over Cache: t=10ms
    
    alt Cache Hit (95%)
        Cache->>Resource: User Info<br/>{user_id: "user-123", scope: "read"}
        Note over Resource: t=15ms
        Resource->>App: Return Data
    else Cache Miss (5%)
        Cache->>Resource: Cache Miss
        Note over Resource: t=20ms
        Resource->>SSO: Validate JWT Token<br/>Verify signature, expiration
        Note over SSO: t=25ms
        SSO->>SSO: Verify RS256 Signature<br/>Using public key
        Note over SSO: t=30ms
        SSO->>SSO: Check Expiration<br/>exp claim
        Note over SSO: t=35ms
        SSO->>SSO: Validate Claims<br/>iss, aud, sub
        Note over SSO: t=40ms
        SSO->>Blacklist: Check Token Blacklist<br/>GET blacklist:eyJ...
        Blacklist->>SSO: Not Revoked
        Note over SSO: t=45ms
        SSO->>Cache: Store Token → User Mapping<br/>SET token:eyJ... {user_info} EX 900
        SSO->>Resource: User Info<br/>{user_id: "user-123", scope: "read"}
        Note over Resource: t=50ms
        Resource->>App: Return Data
    end
```

---

## Token Refresh Flow

**Flow:**

Shows the token refresh flow when access token expires, including refresh token validation and token rotation.

**Steps:**

1. **Access Token Expired** (0ms): Application receives 401 Unauthorized (token expired)
2. **Application Uses Refresh Token** (10ms): Application sends refresh_token to SSO /token endpoint
3. **SSO Validates Refresh Token** (20ms): SSO checks database for refresh token, validates expiration
4. **Check Token Revoked** (30ms): SSO checks if refresh token is revoked
5. **Token Rotation** (40ms): SSO generates new refresh_token, invalidates old one
6. **Issue New Tokens** (50ms): SSO issues new access_token (JWT), new refresh_token, id_token
7. **Application Updates Tokens** (60ms): Application stores new tokens
8. **Retry Original Request** (70ms): Application retries original request with new access_token
9. **Request Succeeds** (80ms): Resource server validates new token, grants access

**Performance:**

- Total latency: <100ms
- Token rotation: <50ms
- Prevents token theft (old refresh token invalidated)

**Security:**

- Refresh token rotation (new token on each use)
- Old refresh token invalidated immediately
- Detects token theft (if old token used, it's revoked)

```mermaid
sequenceDiagram
    participant App as Application
    participant Resource as Resource Server
    participant SSO as SSO Provider
    participant DB as Identity Database
    participant Cache as Redis Cache
    
    App->>Resource: GET /api/data<br/>Authorization: Bearer expired_token
    Note over Resource: t=0ms
    Resource->>App: 401 Unauthorized<br/>Token Expired
    Note over App: t=10ms
    App->>SSO: POST /token<br/>{grant_type: "refresh_token", refresh_token: "..."}
    Note over SSO: t=20ms
    SSO->>DB: SELECT * FROM refresh_tokens<br/>WHERE token_hash = ... AND revoked = false
    Note over DB: t=30ms
    DB->>SSO: Refresh Token Record<br/>{user_id: "user-123", expires_at: "..."}
    SSO->>SSO: Check Expiration<br/>expires_at > now()
    SSO->>SSO: Generate New Refresh Token<br/>random 32 bytes
    Note over SSO: t=40ms
    SSO->>DB: UPDATE refresh_tokens<br/>SET revoked = true WHERE token_id = ...
    SSO->>DB: INSERT refresh_tokens<br/>{new_token_hash, user_id, expires_at}
    SSO->>SSO: Issue New Tokens<br/>Access Token (JWT), New Refresh Token, ID Token
    Note over SSO: t=50ms
    SSO->>App: {access_token: "eyJ...", refresh_token: "new_token", id_token: "eyJ..."}
    Note over App: t=60ms
    App->>App: Store New Tokens<br/>Update HttpOnly cookies
    App->>Resource: GET /api/data<br/>Authorization: Bearer new_token
    Note over Resource: t=70ms
    Resource->>Cache: Validate New Token
    Cache->>Resource: User Info
    Resource->>App: Return Data
    Note over Resource: t=80ms
```

---

## Social Login Flow (Google OAuth)

**Flow:**

Shows the social login flow using Google OAuth, including OAuth redirect, user consent, and account linking.

**Steps:**

1. **User Clicks "Login with Google"** (0ms): User selects Google as identity provider
2. **SSO Redirects to Google** (10ms): SSO redirects user to Google OAuth endpoint
3. **Google Shows Consent Screen** (100ms): Google displays consent screen to user
4. **User Grants Permission** (2000ms): User clicks "Allow" (user time)
5. **Google Redirects with Code** (2100ms): Google redirects to SSO callback with authorization code
6. **SSO Exchanges Code** (2110ms): SSO exchanges code for access token with Google
7. **SSO Gets User Info** (2120ms): SSO uses access token to get user profile from Google
8. **SSO Creates/Links Account** (2150ms): SSO creates local account or links to existing account
9. **SSO Issues Tokens** (2200ms): SSO issues access_token, refresh_token, id_token
10. **Application Receives Tokens** (2250ms): Application receives tokens, user can access resources

**Performance:**

- Total latency: ~2.3 seconds (mostly user time)
- Google OAuth: <200ms
- Account creation/linking: <50ms

**Key Points:**

- No password required (uses Google account)
- Account linking (if user already has SSO account)
- JIT provisioning (create account on first login)

```mermaid
sequenceDiagram
    participant User
    participant App as Application
    participant SSO as SSO Provider
    participant Google as Google OAuth
    participant DB as Identity Database
    
    User->>App: Click "Login with Google"
    Note over App: t=0ms
    App->>SSO: Redirect to /oauth/google
    Note over SSO: t=10ms
    SSO->>Google: Redirect to Google OAuth<br/>?client_id=...&redirect_uri=...
    Note over Google: t=100ms
    Google->>User: Show Consent Screen<br/>"SSO wants to access your profile"
    User->>Google: Click "Allow"
    Note over User: t=2000ms (user time)
    Google->>SSO: Redirect with Code<br/>?code=google_code_123
    Note over SSO: t=2100ms
    SSO->>Google: POST /token<br/>{code: "google_code_123", client_secret: "..."}
    Google->>SSO: {access_token: "google_token", expires_in: 3600}
    Note over SSO: t=2110ms
    SSO->>Google: GET /userinfo<br/>Authorization: Bearer google_token
    Google->>SSO: User Profile<br/>{email: "user@gmail.com", name: "John Doe"}
    Note over SSO: t=2120ms
    SSO->>DB: SELECT * FROM users<br/>WHERE email = "user@gmail.com"
    DB->>SSO: User Not Found
    
    alt Account Exists
        SSO->>SSO: Link Google Account<br/>UPDATE users SET google_id = ...
    else New Account
        SSO->>DB: INSERT INTO users<br/>{email, name, google_id, provider: "google"}
        Note over SSO: t=2150ms
    end
    
    SSO->>SSO: Issue SSO Tokens<br/>Access Token (JWT), Refresh Token, ID Token
    Note over SSO: t=2200ms
    SSO->>App: Redirect with Tokens<br/>?access_token=...&refresh_token=...
    Note over App: t=2250ms
    App->>App: Store Tokens<br/>HttpOnly cookies
    App->>User: Login Successful
```

---

## Multi-Factor Authentication (MFA) Flow

**Flow:**

Shows the MFA flow after initial password authentication, including TOTP code validation.

**Steps:**

1. **User Enters Password** (0ms): User enters username + password
2. **SSO Validates Password** (50ms): SSO validates password hash using bcrypt
3. **SSO Checks MFA Enabled** (100ms): SSO checks if user has MFA enabled
4. **SSO Requests MFA Code** (150ms): SSO prompts user for MFA code (TOTP)
5. **User Enters TOTP Code** (5000ms): User opens authenticator app, enters 6-digit code (user time)
6. **SSO Validates TOTP** (5100ms): SSO validates TOTP code using HMAC-SHA1
7. **MFA Valid** (5150ms): TOTP code valid, proceed with authentication
8. **SSO Issues Tokens** (5200ms): SSO issues access_token, refresh_token, id_token
9. **User Authenticated** (5250ms): User can now access applications

**Performance:**

- Total latency: ~5.3 seconds (mostly user time entering TOTP)
- Password validation: <100ms
- TOTP validation: <10ms

**Security:**

- MFA required for sensitive accounts
- TOTP time window: ±1 time step (90 seconds)
- Rate limiting: Max 5 MFA attempts per 15 minutes

```mermaid
sequenceDiagram
    participant User
    participant SSO as SSO Provider
    participant DB as Identity Database
    participant TOTP as TOTP Validator
    
    User->>SSO: Enter Username + Password
    Note over SSO: t=0ms
    SSO->>DB: SELECT * FROM users<br/>WHERE email = ...
    Note over DB: t=50ms
    DB->>SSO: User Record<br/>{password_hash: "...", mfa_enabled: true, mfa_secret: "..."}
    SSO->>SSO: Validate Password<br/>bcrypt.compare(password, password_hash)
    Note over SSO: t=100ms
    SSO->>SSO: Check MFA Enabled<br/>if mfa_enabled == true
    Note over SSO: t=150ms
    SSO->>User: "Enter 6-digit code from authenticator"
    Note over User: t=5000ms (user time)
    User->>SSO: Enter TOTP Code: 847362
    Note over SSO: t=5100ms
    SSO->>TOTP: Validate TOTP Code<br/>HMAC-SHA1(secret, time_step)
    Note over TOTP: t=5100ms
    TOTP->>TOTP: Generate Expected Code<br/>time_step = now() // 30
    TOTP->>TOTP: HMAC-SHA1(mfa_secret, time_step)
    TOTP->>TOTP: Extract 6-digit code
    TOTP->>SSO: Valid: True (code matches)
    Note over SSO: t=5150ms
    
    alt MFA Invalid
        SSO->>User: "Invalid MFA code. Try again."
    else MFA Valid
        SSO->>SSO: Issue Tokens<br/>Access Token (JWT), Refresh Token, ID Token
        Note over SSO: t=5200ms
        SSO->>User: Login Successful<br/>Redirect to application
        Note over SSO: t=5250ms
    end
```

---

## Single Sign-Out (SSO Logout) Flow

**Flow:**

Shows the single sign-out flow, invalidating all sessions and tokens across all applications.

**Steps:**

1. **User Clicks Logout** (0ms): User clicks "Logout" on any application
2. **Application Sends Logout Request** (10ms): Application sends logout request to SSO
3. **SSO Invalidates Refresh Token** (20ms): SSO marks refresh token as revoked in database
4. **SSO Adds to Blacklist** (30ms): SSO adds access token to blacklist (Redis, TTL = token expiration)
5. **SSO Gets All Sessions** (40ms): SSO retrieves all user sessions from database
6. **SSO Notifies Applications** (50ms): SSO sends logout notifications to all applications (WebSocket/HTTP)
7. **Applications Invalidate Sessions** (100ms): Applications invalidate local sessions
8. **SSO Clears Cookies** (150ms): SSO clears HttpOnly cookies
9. **User Logged Out** (200ms): User logged out from all applications

**Performance:**

- Total latency: <200ms
- Token revocation: <50ms
- Application notifications: <100ms

**Key Points:**

- Single sign-out (logout from all apps)
- Token blacklist (immediate revocation)
- Session invalidation across all applications

```mermaid
sequenceDiagram
    participant User
    participant App1 as Application 1
    participant App2 as Application 2
    participant SSO as SSO Provider
    participant DB as Identity Database
    participant Blacklist as Token Blacklist<br/>Redis
    
    User->>App1: Click "Logout"
    Note over App1: t=0ms
    App1->>SSO: POST /logout<br/>{refresh_token: "..."}
    Note over SSO: t=10ms
    SSO->>DB: UPDATE refresh_tokens<br/>SET revoked = true WHERE token_hash = ...
    Note over DB: t=20ms
    SSO->>Blacklist: ADD access_token to blacklist<br/>SET blacklist:token {user_id} EX 900
    Note over Blacklist: t=30ms
    SSO->>DB: SELECT * FROM sessions<br/>WHERE user_id = ...
    Note over DB: t=40ms
    DB->>SSO: All User Sessions<br/>[{app_id: "app1", session_id: "..."}, {app_id: "app2", session_id: "..."}]
    SSO->>App1: WebSocket / HTTP Notification<br/>"User logged out"
    SSO->>App2: WebSocket / HTTP Notification<br/>"User logged out"
    Note over App1,App2: t=50ms
    App1->>App1: Invalidate Local Session<br/>DELETE session from Redis
    App2->>App2: Invalidate Local Session<br/>DELETE session from Redis
    Note over App1,App2: t=100ms
    SSO->>User: Clear HttpOnly Cookies<br/>Set-Cookie access_token= Max-Age=0
    Note over SSO: t=150ms
    SSO->>User: Redirect to Login Page
    Note over SSO: t=200ms
```

---

## Identity Federation Flow (LDAP)

**Flow:**

Shows the identity federation flow with LDAP/Active Directory, including LDAP authentication and account linking.

**Steps:**

1. **User Selects LDAP Login** (0ms): User selects "Login with LDAP"
2. **SSO Shows LDAP Form** (10ms): SSO displays LDAP login form
3. **User Enters LDAP Credentials** (1000ms): User enters LDAP username + password (user time)
4. **SSO Connects to LDAP** (1100ms): SSO connects to LDAP server
5. **SSO Authenticates with LDAP** (1150ms): SSO binds to LDAP with user credentials
6. **LDAP Returns User Attributes** (1200ms): LDAP returns user attributes (email, name, groups)
7. **SSO Creates/Links Account** (1250ms): SSO creates local account or links to existing account
8. **SSO Issues Tokens** (1300ms): SSO issues access_token, refresh_token, id_token
9. **User Authenticated** (1350ms): User can now access applications

**Performance:**

- Total latency: ~1.4 seconds
- LDAP bind: <100ms
- Account creation/linking: <50ms

**Key Points:**

- Enterprise authentication (Active Directory integration)
- Account linking (if user already has SSO account)
- User attributes from LDAP (groups, roles)

```mermaid
sequenceDiagram
    participant User
    participant SSO as SSO Provider
    participant LDAP as LDAP Server<br/>Active Directory
    participant DB as Identity Database
    
    User->>SSO: Select "Login with LDAP"
    Note over SSO: t=0ms
    SSO->>User: Show LDAP Login Form
    Note over User: t=10ms
    User->>SSO: Enter LDAP Username + Password<br/>username: "john.doe", password: "..."
    Note over User: t=1000ms (user time)
    SSO->>LDAP: Connect to LDAP Server<br/>ldap://ldap.company.com:389
    Note over LDAP: t=1100ms
    SSO->>LDAP: Bind with Credentials<br/>ldap_bind("cn=john.doe,ou=users,dc=company,dc=com", password)
    Note over LDAP: t=1150ms
    LDAP->>SSO: Authentication Success<br/>User Attributes: {email: "john@company.com", name: "John Doe", groups: ["engineers", "admins"]}
    Note over SSO: t=1200ms
    SSO->>DB: SELECT * FROM users<br/>WHERE email = "john@company.com"
    DB->>SSO: User Not Found
    
    alt Account Exists
        SSO->>DB: UPDATE users<br/>SET ldap_dn = ..., ldap_groups = ...
    else New Account
        SSO->>DB: INSERT INTO users<br/>{email, name, ldap_dn, ldap_groups, provider: "ldap"}
        Note over SSO: t=1250ms
    end
    
    SSO->>SSO: Issue SSO Tokens<br/>Access Token (JWT), Refresh Token, ID Token
    Note over SSO: t=1300ms
    SSO->>User: Login Successful<br/>Redirect to application
    Note over SSO: t=1350ms
```

