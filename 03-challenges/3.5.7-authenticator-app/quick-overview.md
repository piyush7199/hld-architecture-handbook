# Authenticator App - Quick Overview

## Core Concept

Authenticator apps like Microsoft Authenticator and Google Authenticator generate time-based one-time passwords (TOTP) for multi-factor authentication. The key innovation is **offline operation** - codes are generated entirely on-device using a shared secret and current time, requiring zero internet connectivity.

**How It Works:**
- User and server share a secret key (established during setup)
- Device computes: `HMAC-SHA1(secret, current_time // 30)` → 6-digit code
- Code valid for 30 seconds, then regenerates
- Server validates code using same algorithm (synchronized time)

## Requirements

### Functional Requirements
- Generate TOTP codes offline (no internet required)
- QR code scanning for account setup
- Multi-device synchronization (phone, tablet, watch)
- Cloud backup for account recovery
- Push notification approvals (Microsoft Authenticator style)
- Account organization (groups, search, favorites)

### Non-Functional Requirements
- **Offline-First**: 100% offline code generation
- **Security**: Secrets encrypted with device HSM (Hardware Security Module)
- **Zero-Knowledge**: Server never sees plaintext secrets
- **Low Latency**: Code generation <10ms
- **Scalability**: 1B+ users (most operations client-side)

## Components

### 1. TOTP Engine (Client-Side)
- Generates 6-digit codes using RFC 6238 algorithm
- Works completely offline (no network calls)
- Handles time synchronization and clock drift

### 2. Key Storage (Encrypted)
- Stores account secrets encrypted with device HSM
- iOS: Keychain (Secure Enclave)
- Android: Keystore (hardware-backed if available)
- Biometric unlock (Face ID / fingerprint)

### 3. Sync Service (Server-Side)
- Syncs account metadata (name, icon, issuer)
- Multi-device synchronization
- Encrypted cloud backup
- **Never syncs secrets** (zero-knowledge)

### 4. Push Notification Service
- Sends approval requests to user's devices
- User taps "Approve" or "Deny"
- Falls back to TOTP codes if push fails
- FCM (Android) / APNs (iOS)

### 5. Account Management
- QR code scanning (otpauth:// URL format)
- Manual secret entry
- Account organization (groups, search)
- Export/import encrypted backups

## Architecture Flow

### Code Generation (Offline)
```
1. User opens app
2. App requests HSM unlock (biometric prompt)
3. HSM returns master key (temporary)
4. App decrypts secrets from database
5. TOTP Engine computes: HMAC-SHA1(secret, time_step)
6. Extract 6-digit code via dynamic truncation
7. Display code to user
8. Code regenerates every 30 seconds
```

### Account Setup (Online)
```
1. Server generates QR code: otpauth://totp/...
2. User scans QR code with app
3. App parses secret + metadata
4. Encrypt secret with HSM master key
5. Store encrypted account in local database
6. (Optional) Upload encrypted account to cloud backup
```

### Background Sync (Online)
```
1. App detects internet connectivity
2. Background service triggers (every 6 hours)
3. App sends sync request: {account_ids, last_sync_times}
4. Server responds: {updates, deleted, new_accounts}
5. App updates local database
6. Sync timestamp updated
```

### Multi-Device Sync
```
Device 1 (Add Account):
1. Encrypt account with device HSM
2. User enables cloud backup → enters password
3. Derive cloud key: PBKDF2(password, salt, 100k iterations)
4. Encrypt account with cloud key
5. Upload encrypted account to cloud

Device 2 (Restore):
1. User installs app, enters cloud password
2. Derive same cloud key
3. Download encrypted accounts from cloud
4. Decrypt with cloud key
5. Re-encrypt with Device 2's HSM master key
6. Store in Device 2's database
```

### Push Approval Flow
```
1. User attempts login on website
2. Website → Authenticator Service: approval request
3. Service → Mobile App: push notification
4. User taps "Approve" on phone
5. App → Service: approval response
6. Service → Website: login approved
7. Website completes authentication
```

## Key Design Decisions

### 1. Offline-First Architecture
**Decision**: Code generation requires zero network calls.

**Why:**
- Works in airplane mode, no signal areas
- No server dependency (100% availability)
- Fast (local computation, <10ms)

**Trade-off**: Account metadata syncs eventually (not real-time).

### 2. Zero-Knowledge Encryption
**Decision**: Server never sees plaintext secrets.

**Why:**
- Server compromise doesn't expose secrets
- User controls encryption (cloud backup password)
- Privacy-first design

**Trade-off**: Can't recover accounts if password forgotten.

### 3. Hardware Security Module (HSM)
**Decision**: Use device HSM (Keychain/Keystore) for key encryption.

**Why:**
- Hardware-backed encryption (tamper-resistant)
- Biometric unlock (convenient security)
- Keys never leave device

**Trade-off**: Requires biometric-capable device.

### 4. Time Window Tolerance
**Decision**: Accept codes from ±1 time step (90-second window).

**Why:**
- Handles device clock drift automatically
- Better user experience (no manual time sync)
- Standard TOTP practice

**Trade-off**: Slightly larger attack window (90s vs 30s).

### 5. Cloud Backup (Optional)
**Decision**: Encrypted backup with user password (no password reset).

**Why:**
- Account recovery if device lost
- Multi-device sync
- User controls encryption key

**Trade-off**: Password forgotten = accounts lost (security feature).

## Bottlenecks & Solutions

### Bottleneck 1: HSM Unlock Latency
**Problem**: Biometric unlock + decryption takes 50-100ms.

**Solution**: Cache decrypted secrets in memory (encrypted with session key).
- First unlock: 100ms (HSM)
- Subsequent codes: <1ms (cached)
- Clear cache when app backgrounded

### Bottleneck 2: Code Generation (Minor)
**Problem**: HMAC-SHA1 computation (~1ms, but can optimize).

**Solution**: Pre-compute codes for next 2 time steps.
- Display cached code (0ms)
- Background computation for next codes

### Bottleneck 3: Sync Service Throughput
**Problem**: 350 QPS sync operations.

**Solution**: Batch sync requests.
- Sync multiple accounts in one request
- Delta sync (only changed accounts)
- Result: 350 QPS → 50 QPS (7x reduction)

### Bottleneck 4: Push Notification Latency
**Problem**: FCM/APNs latency (500ms-2s).

**Solution**: None (depends on Google/Apple infrastructure).
- Fallback: TOTP codes always available
- Push is convenience, not requirement

## Common Anti-Patterns

### ❌ **1. Storing Secrets in Plaintext**
**Problem**: Secrets stored unencrypted in database.
```sql
CREATE TABLE accounts (secret TEXT);  -- BAD!
```

**Solution**: Encrypt with device HSM.
```sql
CREATE TABLE accounts (
    encrypted_secret TEXT,  -- AES-256-GCM
    encryption_nonce TEXT
);
```

### ❌ **2. Sending Secrets to Server**
**Problem**: Uploading plaintext secrets to server.
```javascript
POST /api/accounts {secret: "JBSWY3DPEHPK3PXP"}  // BAD!
```

**Solution**: Encrypt client-side before upload (zero-knowledge).
```javascript
POST /api/accounts {
    encrypted_secret: "AES-256-GCM(secret, cloud_key)"
}
```

### ❌ **3. Generating Codes on Server**
**Problem**: Server generates codes (requires internet).
```javascript
GET /api/code?account_id=123  // BAD! Requires internet
```

**Solution**: Generate codes locally (offline).
```javascript
function generateCode(secret) {
    return generate_totp(secret, Date.now());  // Works offline!
}
```

### ❌ **4. No Time Drift Handling**
**Problem**: Only accepts exact time step (fails with clock drift).
```javascript
return code === generate_totp(secret, Date.now());  // BAD!
```

**Solution**: Accept ±1 time step (90-second window).
```javascript
const steps = [now - 30, now, now + 30];
return steps.some(step => generate_totp(secret, step) === code);
```

### ❌ **5. Weak Cloud Backup Encryption**
**Problem**: Simple encryption without key derivation.
```javascript
const encrypted = AES(account, password);  // BAD! No key derivation
```

**Solution**: PBKDF2 key derivation (100k iterations).
```javascript
const key = PBKDF2(password, salt, 100000, 'SHA-256');
const encrypted = AES-256-GCM(account, key, nonce);
```

## Monitoring & Observability

### Key Metrics

**Client-Side:**
- Code generation latency: <10ms (p99)
- HSM unlock latency: <200ms (p99)
- App crash rate: <0.1%
- Offline usage: % of codes generated offline

**Server-Side:**
- Sync request latency: <100ms (p95)
- Sync success rate: >99.9%
- Push delivery latency: <2s (p95)
- Push delivery rate: >99%
- API error rate: <0.1%

### Alerts

**Critical:**
- Sync service down: >5% error rate for 5 minutes
- Push delivery failure: >10% failure rate for 5 minutes
- Database latency: >500ms (p95) for 10 minutes

**Warning:**
- High sync latency: >200ms (p95) for 30 minutes
- Increased error rate: >1% for 15 minutes

## Trade-offs Summary

| What You Gain | What You Sacrifice |
|---------------|-------------------|
| ✅ Offline Operation | ❌ Eventual consistency (metadata sync) |
| ✅ Zero-Knowledge Security | ❌ No password reset (accounts lost if forgotten) |
| ✅ Hardware Security (HSM) | ❌ Requires biometric-capable device |
| ✅ Multi-Device Sync | ❌ Cloud backup requires user password |
| ✅ Push Notifications | ❌ Requires internet (TOTP fallback available) |
| ✅ Fast Code Generation | ❌ Cached secrets in memory (cleared on background) |
| ✅ Scalable (Client-Side) | ❌ Limited server-side features |

**Key Trade-off**: **Security vs. Convenience**
- More secure: Offline TOTP, zero-knowledge, HSM encryption
- Less convenient: Manual setup, no password reset

## Real-World Examples

### Google Authenticator
- **500M+ users**
- Offline TOTP generation
- No cloud backup (security-focused)
- QR code setup

### Microsoft Authenticator
- **200M+ users**
- Offline TOTP + push notifications
- Encrypted cloud backup
- Multi-device sync

### Authy
- **50M+ users**
- Offline TOTP + cloud backup
- Multi-device sync
- SMS backup codes (optional)

## Key Takeaways

1. **Offline-First**: TOTP codes generated locally (no internet required)
2. **Zero-Knowledge**: Server never sees plaintext secrets
3. **Hardware Security**: Device HSM (Keychain/Keystore) for encryption
4. **Time Tolerance**: Accept ±1 time step (90-second window)
5. **Multi-Device**: Encrypted cloud backup with user password
6. **Push Fallback**: TOTP codes always available if push fails
7. **Scalability**: Most operations client-side (minimal server infrastructure)

## Recommended Stack

**Client:**
- iOS: Swift, Keychain (Secure Enclave), APNs
- Android: Kotlin, Keystore (hardware-backed), FCM

**Server:**
- Sync Service: Node.js/Python, PostgreSQL, Redis
- Push Service: FCM (Android), APNs (iOS), RabbitMQ

**Security:**
- Encryption: AES-256-GCM (at rest), TLS 1.3 (in transit)
- Key Derivation: PBKDF2 (100k iterations, SHA-256)
- Key Management: AWS KMS / Azure Key Vault

