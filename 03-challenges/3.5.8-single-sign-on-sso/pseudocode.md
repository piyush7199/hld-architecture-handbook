# Single Sign-On (SSO) System - Pseudocode Implementations

This document contains detailed algorithm implementations for the Single Sign-On (SSO) system. The main challenge document references these functions.

---

## Table of Contents

1. [OAuth 2.0 Authorization Code Flow](#oauth-20-authorization-code-flow)
2. [SAML 2.0 SSO Flow](#saml-20-sso-flow)
3. [Token Management](#token-management)
4. [Password Hashing and Verification](#password-hashing-and-verification)
5. [Token Validation](#token-validation)
6. [Identity Federation](#identity-federation)
7. [Multi-Factor Authentication](#multi-factor-authentication)

---

## OAuth 2.0 Authorization Code Flow

### oauth_authorization_code_flow()

Handles the OAuth 2.0 Authorization Code Flow, from authorization request to token issuance.

**Parameters:**

- `client_id` (string): Application client ID
- `redirect_uri` (string): Application redirect URI
- `scope` (string, optional): OAuth scopes (default: "openid profile email")
- `state` (string, optional): State parameter for CSRF protection

**Returns:**

- `object`: Authorization response with authorization_code or error

**Algorithm:**

```
function oauth_authorization_code_flow(client_id, redirect_uri, scope = "openid profile email", state = None):
    // Validate client
    application = database.find_application(client_id)
    if not application:
        throw InvalidClientException("Invalid client_id")
    
    // Validate redirect_uri
    if redirect_uri not in application.redirect_uris:
        throw InvalidRedirectURIException("Redirect URI not registered")
    
    // Check if user is authenticated
    if not user_authenticated():
        // Show login form
        return {
            "action": "login_required",
            "client_id": client_id,
            "redirect_uri": redirect_uri,
            "scope": scope,
            "state": state
        }
    
    // User is authenticated, generate authorization code
    user = get_current_user()
    
    // Generate random authorization code (32 bytes)
    authorization_code = generate_random_bytes(32)
    code_string = base64url_encode(authorization_code)
    
    // Store code → user mapping in Redis (10-minute TTL)
    redis.set(
        key = "code:" + code_string,
        value = json_encode({
            "user_id": user.user_id,
            "client_id": client_id,
            "scope": scope,
            "redirect_uri": redirect_uri
        }),
        ttl = 600  // 10 minutes
    )
    
    // Build redirect URL
    redirect_url = redirect_uri + "?code=" + code_string
    if state:
        redirect_url = redirect_url + "&state=" + state
    
    return {
        "action": "redirect",
        "redirect_url": redirect_url
    }
```

**Time Complexity:** O(1)

**Example Usage:**

```
response = oauth_authorization_code_flow(
    client_id = "app-123",
    redirect_uri = "https://app.example.com/callback",
    scope = "openid profile email",
    state = "random-state-123"
)
```

### exchange_authorization_code()

Exchanges authorization code for access token, refresh token, and ID token.

**Parameters:**

- `code` (string): Authorization code
- `client_id` (string): Application client ID
- `client_secret` (string): Application client secret
- `redirect_uri` (string): Application redirect URI

**Returns:**

- `object`: Token response with access_token, refresh_token, id_token

**Algorithm:**

```
function exchange_authorization_code(code, client_id, client_secret, redirect_uri):
    // Validate client
    application = database.find_application(client_id)
    if not application:
        throw InvalidClientException("Invalid client_id")
    
    // Validate client_secret
    if not bcrypt.compare(client_secret, application.client_secret_hash):
        throw InvalidClientException("Invalid client_secret")
    
    // Retrieve code from Redis
    code_data = redis.get("code:" + code)
    if not code_data:
        throw InvalidCodeException("Authorization code expired or invalid")
    
    code_info = json_decode(code_data)
    
    // Validate redirect_uri matches
    if code_info.redirect_uri != redirect_uri:
        throw InvalidRedirectURIException("Redirect URI mismatch")
    
    // Validate client_id matches
    if code_info.client_id != client_id:
        throw InvalidClientException("Client ID mismatch")
    
    // Delete code from Redis (one-time use)
    redis.delete("code:" + code)
    
    // Get user
    user = database.find_user(code_info.user_id)
    
    // Generate access token (JWT)
    access_token = generate_access_token(
        user_id = user.user_id,
        email = user.email,
        scope = code_info.scope,
        client_id = client_id,
        expires_in = 900  // 15 minutes
    )
    
    // Generate refresh token
    refresh_token = generate_refresh_token(
        user_id = user.user_id,
        client_id = client_id,
        expires_in = 2592000  // 30 days
    )
    
    // Generate ID token (OIDC)
    id_token = generate_id_token(
        user_id = user.user_id,
        email = user.email,
        name = user.name,
        client_id = client_id,
        expires_in = 900  // 15 minutes
    )
    
    return {
        "access_token": access_token,
        "token_type": "Bearer",
        "expires_in": 900,
        "refresh_token": refresh_token,
        "id_token": id_token,
        "scope": code_info.scope
    }
```

**Time Complexity:** O(1)

---

## SAML 2.0 SSO Flow

### saml_sso_flow()

Handles the SAML 2.0 SSO flow, including SAML request parsing and assertion generation.

**Parameters:**

- `saml_request` (string): Base64-encoded SAML AuthnRequest (XML)
- `relay_state` (string, optional): Relay state parameter

**Returns:**

- `object`: SAML response with SAMLResponse (XML) or login form

**Algorithm:**

```
function saml_sso_flow(saml_request, relay_state = None):
    // Decode SAML request
    saml_xml = base64_decode(saml_request)
    
    // Parse SAML XML
    saml_doc = parse_xml(saml_xml)
    
    // Extract SP entity ID and ACS URL
    sp_entity_id = saml_doc.get_attribute("Issuer")
    acs_url = saml_doc.get_attribute("AssertionConsumerServiceURL")
    
    // Validate SP (Service Provider)
    sp = database.find_service_provider(sp_entity_id)
    if not sp:
        throw InvalidSPException("Service Provider not registered")
    
    // Validate ACS URL
    if acs_url not in sp.acs_urls:
        throw InvalidACSUrlException("ACS URL not registered")
    
    // Check if user is authenticated
    if not user_authenticated():
        // Show login form
        return {
            "action": "login_required",
            "saml_request": saml_request,
            "relay_state": relay_state,
            "sp_entity_id": sp_entity_id
        }
    
    // User is authenticated, generate SAML assertion
    user = get_current_user()
    
    // Generate SAML assertion (XML)
    assertion = generate_saml_assertion(
        user = user,
        sp_entity_id = sp_entity_id,
        acs_url = acs_url,
        issuer = "https://sso.example.com",
        not_before = current_timestamp() - 60,  // 1 minute before
        not_on_or_after = current_timestamp() + 300  // 5 minutes after
    )
    
    // Sign assertion with private key
    signed_assertion = sign_saml_assertion(assertion, private_key)
    
    // Create SAML response
    saml_response = create_saml_response(
        assertion = signed_assertion,
        in_response_to = saml_doc.get_attribute("ID"),
        destination = acs_url
    )
    
    // Encode SAML response (base64)
    saml_response_encoded = base64_encode(saml_response)
    
    return {
        "action": "redirect",
        "acs_url": acs_url,
        "saml_response": saml_response_encoded,
        "relay_state": relay_state
    }
```

**Time Complexity:** O(n) where n is XML size

### generate_saml_assertion()

Generates a SAML 2.0 assertion with user attributes.

**Parameters:**

- `user` (object): User object
- `sp_entity_id` (string): Service Provider entity ID
- `acs_url` (string): Assertion Consumer Service URL
- `issuer` (string): Identity Provider issuer
- `not_before` (integer): Assertion not valid before (Unix timestamp)
- `not_on_or_after` (integer): Assertion not valid after (Unix timestamp)

**Returns:**

- `string`: SAML assertion XML

**Algorithm:**

```
function generate_saml_assertion(user, sp_entity_id, acs_url, issuer, not_before, not_on_or_after):
    assertion_id = generate_uuid()
    issue_instant = current_timestamp_iso8601()
    
    // Build SAML assertion XML
    assertion_xml = """
    <saml:Assertion xmlns:saml="urn:oasis:names:tc:SAML:2.0:assertion"
                    ID="{assertion_id}"
                    IssueInstant="{issue_instant}"
                    Version="2.0">
        <saml:Issuer>{issuer}</saml:Issuer>
        <saml:Subject>
            <saml:NameID Format="urn:oasis:names:tc:SAML:1.1:nameid-format:emailAddress">
                {user.email}
            </saml:NameID>
            <saml:SubjectConfirmation Method="urn:oasis:names:tc:SAML:2.0:cm:bearer">
                <saml:SubjectConfirmationData NotOnOrAfter="{not_on_or_after}"
                                               Recipient="{acs_url}"/>
            </saml:SubjectConfirmation>
        </saml:Subject>
        <saml:Conditions NotBefore="{not_before}" NotOnOrAfter="{not_on_or_after}">
            <saml:AudienceRestriction>
                <saml:Audience>{sp_entity_id}</saml:Audience>
            </saml:AudienceRestriction>
        </saml:Conditions>
        <saml:AuthnStatement AuthnInstant="{issue_instant}">
            <saml:AuthnContext>
                <saml:AuthnContextClassRef>
                    urn:oasis:names:tc:SAML:2.0:ac:classes:PasswordProtectedTransport
                </saml:AuthnContextClassRef>
            </saml:AuthnContext>
        </saml:AuthnStatement>
        <saml:AttributeStatement>
            <saml:Attribute Name="email">
                <saml:AttributeValue>{user.email}</saml:AttributeValue>
            </saml:Attribute>
            <saml:Attribute Name="name">
                <saml:AttributeValue>{user.name}</saml:AttributeValue>
            </saml:Attribute>
        </saml:AttributeStatement>
    </saml:Assertion>
    """
    
    return format(assertion_xml, 
                  assertion_id = assertion_id,
                  issue_instant = issue_instant,
                  issuer = issuer,
                  user_email = user.email,
                  user_name = user.name,
                  not_before = timestamp_to_iso8601(not_before),
                  not_on_or_after = timestamp_to_iso8601(not_on_or_after),
                  acs_url = acs_url,
                  sp_entity_id = sp_entity_id)
```

**Time Complexity:** O(1)

---

## Token Management

### generate_access_token()

Generates a JWT access token with user claims.

**Parameters:**

- `user_id` (string): User ID
- `email` (string): User email
- `scope` (string): OAuth scopes
- `client_id` (string): Application client ID
- `expires_in` (integer): Token expiration in seconds (default: 900)

**Returns:**

- `string`: JWT access token

**Algorithm:**

```
function generate_access_token(user_id, email, scope, client_id, expires_in = 900):
    now = current_timestamp()
    
    // Build JWT payload
    payload = {
        "sub": user_id,  // Subject (user ID)
        "email": email,
        "scope": scope,
        "aud": client_id,  // Audience (client ID)
        "iss": "https://sso.example.com",  // Issuer
        "iat": now,  // Issued at
        "exp": now + expires_in,  // Expiration
        "jti": generate_uuid()  // JWT ID (for blacklisting)
    }
    
    // Sign JWT with RS256 (RSA) or HS256 (HMAC)
    header = {
        "alg": "RS256",
        "typ": "JWT"
    }
    
    // Encode header and payload
    header_encoded = base64url_encode(json_encode(header))
    payload_encoded = base64url_encode(json_encode(payload))
    
    // Create signature
    signature_input = header_encoded + "." + payload_encoded
    signature = rsa_sign(signature_input, private_key, "SHA256")
    signature_encoded = base64url_encode(signature)
    
    // Build JWT
    jwt = header_encoded + "." + payload_encoded + "." + signature_encoded
    
    // Store token → user mapping in Redis cache (TTL = expires_in)
    redis.set(
        key = "token:" + jwt,
        value = json_encode({
            "user_id": user_id,
            "email": email,
            "scope": scope
        }),
        ttl = expires_in
    )
    
    return jwt
```

**Time Complexity:** O(1)

### generate_refresh_token()

Generates a refresh token and stores it in the database.

**Parameters:**

- `user_id` (string): User ID
- `client_id` (string): Application client ID
- `expires_in` (integer): Token expiration in seconds (default: 2592000)

**Returns:**

- `string`: Refresh token

**Algorithm:**

```
function generate_refresh_token(user_id, client_id, expires_in = 2592000):
    // Generate random refresh token (32 bytes)
    refresh_token_bytes = generate_random_bytes(32)
    refresh_token = base64url_encode(refresh_token_bytes)
    
    // Hash refresh token (for storage)
    refresh_token_hash = sha256_hash(refresh_token)
    
    // Calculate expiration
    expires_at = current_timestamp() + expires_in
    
    // Store in database
    database.insert_refresh_token({
        "token_hash": refresh_token_hash,
        "user_id": user_id,
        "client_id": client_id,
        "expires_at": expires_at,
        "revoked": false,
        "created_at": current_timestamp()
    })
    
    return refresh_token
```

**Time Complexity:** O(1)

### refresh_access_token()

Refreshes an access token using a refresh token, with token rotation.

**Parameters:**

- `refresh_token` (string): Refresh token
- `client_id` (string): Application client ID
- `client_secret` (string): Application client secret

**Returns:**

- `object`: New access token, refresh token, and ID token

**Algorithm:**

```
function refresh_access_token(refresh_token, client_id, client_secret):
    // Validate client
    application = database.find_application(client_id)
    if not application:
        throw InvalidClientException("Invalid client_id")
    
    // Validate client_secret
    if not bcrypt.compare(client_secret, application.client_secret_hash):
        throw InvalidClientException("Invalid client_secret")
    
    // Hash refresh token
    refresh_token_hash = sha256_hash(refresh_token)
    
    // Find refresh token in database
    token_record = database.find_refresh_token(refresh_token_hash)
    if not token_record:
        throw InvalidTokenException("Refresh token not found")
    
    // Check if revoked
    if token_record.revoked:
        throw InvalidTokenException("Refresh token revoked")
    
    // Check if expired
    if token_record.expires_at < current_timestamp():
        throw InvalidTokenException("Refresh token expired")
    
    // Check client_id matches
    if token_record.client_id != client_id:
        throw InvalidClientException("Client ID mismatch")
    
    // Get user
    user = database.find_user(token_record.user_id)
    
    // Revoke old refresh token (token rotation)
    database.update_refresh_token(
        token_id = token_record.token_id,
        revoked = true
    )
    
    // Generate new access token
    access_token = generate_access_token(
        user_id = user.user_id,
        email = user.email,
        scope = "openid profile email",  // Default scope
        client_id = client_id,
        expires_in = 900
    )
    
    // Generate new refresh token
    new_refresh_token = generate_refresh_token(
        user_id = user.user_id,
        client_id = client_id,
        expires_in = 2592000
    )
    
    // Generate new ID token
    id_token = generate_id_token(
        user_id = user.user_id,
        email = user.email,
        name = user.name,
        client_id = client_id,
        expires_in = 900
    )
    
    return {
        "access_token": access_token,
        "token_type": "Bearer",
        "expires_in": 900,
        "refresh_token": new_refresh_token,
        "id_token": id_token
    }
```

**Time Complexity:** O(1)

---

## Password Hashing and Verification

### hash_password()

Hashes a password using bcrypt.

**Parameters:**

- `password` (string): Plaintext password

**Returns:**

- `string`: bcrypt hash

**Algorithm:**

```
function hash_password(password):
    // Generate random salt
    salt = bcrypt_gensalt(cost_factor = 10)
    
    // Hash password with salt
    password_hash = bcrypt_hash(password, salt)
    
    return password_hash
```

**Time Complexity:** O(1) (but ~100ms due to bcrypt cost factor)

### verify_password()

Verifies a password against a bcrypt hash.

**Parameters:**

- `password` (string): Plaintext password
- `password_hash` (string): bcrypt hash

**Returns:**

- `boolean`: True if password matches, False otherwise

**Algorithm:**

```
function verify_password(password, password_hash):
    // Constant-time comparison (prevents timing attacks)
    return bcrypt_compare(password, password_hash)
```

**Time Complexity:** O(1) (but ~100ms due to bcrypt cost factor)

---

## Token Validation

### validate_access_token()

Validates a JWT access token, checking signature, expiration, and blacklist.

**Parameters:**

- `access_token` (string): JWT access token

**Returns:**

- `object`: User info if valid, None if invalid

**Algorithm:**

```
function validate_access_token(access_token):
    // Check Redis cache first (95% hit rate)
    cached_user = redis.get("token:" + access_token)
    if cached_user:
        return json_decode(cached_user)  // Cache hit, return immediately
    
    // Cache miss, validate JWT
    // Split JWT into parts
    parts = access_token.split(".")
    if len(parts) != 3:
        throw InvalidTokenException("Invalid JWT format")
    
    header_encoded = parts[0]
    payload_encoded = parts[1]
    signature_encoded = parts[2]
    
    // Decode header
    header = json_decode(base64url_decode(header_encoded))
    
    // Decode payload
    payload = json_decode(base64url_decode(payload_encoded))
    
    // Verify signature
    signature_input = header_encoded + "." + payload_encoded
    signature = base64url_decode(signature_encoded)
    
    if header.alg == "RS256":
        public_key = get_public_key(payload.iss)
        if not rsa_verify(signature_input, signature, public_key, "SHA256"):
            throw InvalidTokenException("Invalid signature")
    else if header.alg == "HS256":
        secret_key = get_secret_key(payload.iss)
        expected_signature = hmac_sha256(signature_input, secret_key)
        if not constant_time_compare(signature, expected_signature):
            throw InvalidTokenException("Invalid signature")
    
    // Check expiration
    now = current_timestamp()
    if payload.exp < now:
        throw InvalidTokenException("Token expired")
    
    // Check issuer
    if payload.iss != "https://sso.example.com":
        throw InvalidTokenException("Invalid issuer")
    
    // Check token blacklist
    if redis.exists("blacklist:" + access_token):
        throw InvalidTokenException("Token revoked")
    
    // Extract user info
    user_info = {
        "user_id": payload.sub,
        "email": payload.email,
        "scope": payload.scope
    }
    
    // Store in cache (TTL = remaining expiration time)
    remaining_ttl = payload.exp - now
    if remaining_ttl > 0:
        redis.set(
            key = "token:" + access_token,
            value = json_encode(user_info),
            ttl = remaining_ttl
        )
    
    return user_info
```

**Time Complexity:** O(1) (cache hit) or O(1) (JWT validation, but ~50ms)

---

## Identity Federation

### federate_identity()

Federates identity with external provider (Google, Microsoft, LDAP).

**Parameters:**

- `provider` (string): Identity provider ("google", "microsoft", "ldap")
- `authorization_code` (string, optional): OAuth authorization code (for Google/Microsoft)
- `username` (string, optional): LDAP username
- `password` (string, optional): LDAP password

**Returns:**

- `object`: User info from external provider

**Algorithm:**

```
function federate_identity(provider, authorization_code = None, username = None, password = None):
    if provider == "google":
        return federate_google(authorization_code)
    else if provider == "microsoft":
        return federate_microsoft(authorization_code)
    else if provider == "ldap":
        return federate_ldap(username, password)
    else:
        throw InvalidProviderException("Unsupported provider")

function federate_google(authorization_code):
    // Exchange code for access token
    token_response = http_post("https://oauth2.googleapis.com/token", {
        "code": authorization_code,
        "client_id": google_client_id,
        "client_secret": google_client_secret,
        "redirect_uri": google_redirect_uri,
        "grant_type": "authorization_code"
    })
    
    access_token = token_response.access_token
    
    // Get user info
    user_response = http_get("https://www.googleapis.com/oauth2/v2/userinfo", {
        "Authorization": "Bearer " + access_token
    })
    
    return {
        "email": user_response.email,
        "name": user_response.name,
        "provider": "google",
        "provider_id": user_response.id
    }

function federate_ldap(username, password):
    // Connect to LDAP server
    ldap_conn = ldap_connect("ldap://ldap.company.com:389")
    
    // Bind with credentials
    ldap_conn.bind("cn=" + username + ",ou=users,dc=company,dc=com", password)
    
    // Search for user
    user_dn = "cn=" + username + ",ou=users,dc=company,dc=com"
    user_attrs = ldap_conn.search(user_dn, attributes = ["mail", "cn", "memberOf"])
    
    return {
        "email": user_attrs.mail,
        "name": user_attrs.cn,
        "provider": "ldap",
        "provider_id": user_dn,
        "groups": user_attrs.memberOf
    }
```

**Time Complexity:** O(1) (network calls add latency)

---

## Multi-Factor Authentication

### verify_mfa()

Verifies a multi-factor authentication code (TOTP).

**Parameters:**

- `user_id` (string): User ID
- `mfa_code` (string): TOTP code (6 digits)

**Returns:**

- `boolean`: True if MFA code is valid, False otherwise

**Algorithm:**

```
function verify_mfa(user_id, mfa_code):
    // Get user
    user = database.find_user(user_id)
    if not user:
        throw UserNotFoundException("User not found")
    
    // Check if MFA enabled
    if not user.mfa_enabled:
        throw MFANotEnabledException("MFA not enabled for user")
    
    // Get MFA secret (decrypt if encrypted)
    mfa_secret = decrypt_secret(user.mfa_secret, hsm_master_key)
    
    // Get current time
    now = current_timestamp()
    
    // Try current time step
    time_step = now / 30  // 30-second window
    expected_code = generate_totp(mfa_secret, time_step)
    if mfa_code == expected_code:
        return true
    
    // Try previous time step (-30 seconds)
    prev_time_step = (now - 30) / 30
    prev_code = generate_totp(mfa_secret, prev_time_step)
    if mfa_code == prev_code:
        return true
    
    // Try next time step (+30 seconds)
    next_time_step = (now + 30) / 30
    next_code = generate_totp(mfa_secret, next_time_step)
    if mfa_code == next_code:
        return true
    
    // Code not valid in any time window
    return false

function generate_totp(secret, time_step):
    // Decode Base32 secret
    secret_bytes = base32_decode(secret)
    
    // Convert time step to 8-byte big-endian
    time_step_bytes = int_to_bytes_big_endian(time_step, 8)
    
    // Compute HMAC-SHA1
    hmac_hash = hmac_sha1(secret_bytes, time_step_bytes)
    
    // Dynamic truncation
    offset = hmac_hash[19] & 0x0F
    binary_code = (hmac_hash[offset] & 0x7F) << 24
    binary_code = binary_code | ((hmac_hash[offset + 1] & 0xFF) << 16)
    binary_code = binary_code | ((hmac_hash[offset + 2] & 0xFF) << 8)
    binary_code = binary_code | (hmac_hash[offset + 3] & 0xFF)
    
    // Generate 6-digit code
    code = binary_code % 1000000
    
    // Format as string with leading zeros
    return format(code, "06d")
```

**Time Complexity:** O(1)

### create_session()

Creates a user session (for stateful session management).

**Parameters:**

- `user_id` (string): User ID
- `tenant_id` (string): Tenant ID
- `expires_in` (integer): Session expiration in seconds (default: 3600)

**Returns:**

- `string`: Session ID

**Algorithm:**

```
function create_session(user_id, tenant_id, expires_in = 3600):
    // Generate session ID
    session_id = generate_uuid()
    
    // Calculate expiration
    expires_at = current_timestamp() + expires_in
    
    // Store session in Redis
    redis.set(
        key = "session:" + session_id,
        value = json_encode({
            "user_id": user_id,
            "tenant_id": tenant_id,
            "created_at": current_timestamp(),
            "expires_at": expires_at
        }),
        ttl = expires_in
    )
    
    // Also store in database (for persistence)
    database.insert_session({
        "session_id": session_id,
        "user_id": user_id,
        "tenant_id": tenant_id,
        "expires_at": expires_at,
        "created_at": current_timestamp()
    })
    
    return session_id
```

**Time Complexity:** O(1)

### validate_session()

Validates a session ID and returns user info.

**Parameters:**

- `session_id` (string): Session ID

**Returns:**

- `object`: User info if valid, None if invalid

**Algorithm:**

```
function validate_session(session_id):
    // Check Redis cache first
    cached_session = redis.get("session:" + session_id)
    if cached_session:
        session_data = json_decode(cached_session)
        // Check expiration
        if session_data.expires_at < current_timestamp():
            redis.delete("session:" + session_id)
            return None
        return {
            "user_id": session_data.user_id,
            "tenant_id": session_data.tenant_id
        }
    
    // Cache miss, check database
    session_record = database.find_session(session_id)
    if not session_record:
        return None
    
    // Check expiration
    if session_record.expires_at < current_timestamp():
        database.delete_session(session_id)
        return None
    
    // Get user info
    user = database.find_user(session_record.user_id)
    if not user:
        return None
    
    // Store in cache
    remaining_ttl = session_record.expires_at - current_timestamp()
    if remaining_ttl > 0:
        redis.set(
            key = "session:" + session_id,
            value = json_encode({
                "user_id": user.user_id,
                "tenant_id": session_record.tenant_id,
                "expires_at": session_record.expires_at
            }),
            ttl = remaining_ttl
        )
    
    return {
        "user_id": user.user_id,
        "tenant_id": session_record.tenant_id,
        "email": user.email
    }
```

**Time Complexity:** O(1)

### generate_id_token()

Generates an OpenID Connect ID token (JWT).

**Parameters:**

- `user_id` (string): User ID
- `email` (string): User email
- `name` (string): User name
- `client_id` (string): Application client ID
- `expires_in` (integer): Token expiration in seconds (default: 900)

**Returns:**

- `string`: JWT ID token

**Algorithm:**

```
function generate_id_token(user_id, email, name, client_id, expires_in = 900):
    now = current_timestamp()
    
    // Build ID token payload (OIDC standard claims)
    payload = {
        "sub": user_id,  // Subject (user ID)
        "email": email,
        "name": name,
        "email_verified": true,
        "aud": client_id,  // Audience (client ID)
        "iss": "https://sso.example.com",  // Issuer
        "iat": now,  // Issued at
        "exp": now + expires_in,  // Expiration
        "auth_time": now  // Authentication time
    }
    
    // Sign JWT (same as access token)
    header = {
        "alg": "RS256",
        "typ": "JWT"
    }
    
    header_encoded = base64url_encode(json_encode(header))
    payload_encoded = base64url_encode(json_encode(payload))
    
    signature_input = header_encoded + "." + payload_encoded
    signature = rsa_sign(signature_input, private_key, "SHA256")
    signature_encoded = base64url_encode(signature)
    
    id_token = header_encoded + "." + payload_encoded + "." + signature_encoded
    
    return id_token
```

**Time Complexity:** O(1)

---

## Summary

All functions implement the core SSO algorithms:

- **OAuth 2.0**: Authorization code flow, token exchange
- **SAML 2.0**: SSO flow, assertion generation
- **Token Management**: JWT generation, refresh token rotation
- **Password Security**: bcrypt hashing and verification
- **Token Validation**: JWT validation with caching
- **Identity Federation**: Google, Microsoft, LDAP integration
- **MFA**: TOTP verification with time window tolerance

