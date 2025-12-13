# Authenticator App - Design Decisions (This Over That)

This document provides in-depth analysis of all major architectural decisions made when designing an authenticator app like Microsoft Authenticator or Google Authenticator.

---

## Table of Contents

1. [Offline-First vs Online-First Architecture](#1-offline-first-vs-online-first-architecture)
2. [TOTP vs HOTP vs SMS OTP](#2-totp-vs-hotp-vs-sms-otp)
3. [Device HSM vs Software Encryption](#3-device-hsm-vs-software-encryption)
4. [Cloud Backup vs Local-Only Storage](#4-cloud-backup-vs-local-only-storage)
5. [Push Notifications vs TOTP-Only](#5-push-notifications-vs-totp-only)
6. [Synchronous vs Asynchronous Sync](#6-synchronous-vs-asynchronous-sync)
7. [SQLite vs NoSQL for Local Storage](#7-sqlite-vs-nosql-for-local-storage)
8. [PBKDF2 vs Argon2 for Key Derivation](#8-pbkdf2-vs-argon2-for-key-derivation)

---

## 1. Offline-First vs Online-First Architecture

### The Problem

Authenticator apps must generate codes even when devices have no internet connectivity (airplane mode, no signal, network outages).

### Options Considered

| Feature | Offline-First | Online-First |
|---------|---------------|--------------|
| **Code Generation** | ✅ Works offline (100% availability) | ❌ Requires internet |
| **Server Load** | ✅ Minimal (only sync/push) | ❌ High (every code generation) |
| **Latency** | ✅ <10ms (local computation) | ❌ 100-500ms (network round-trip) |
| **Availability** | ✅ 100% (no server dependency) | ❌ 99.9% (server-dependent) |
| **Scalability** | ✅ Infinite (client-side) | ⚠️ Limited by server capacity |
| **Complexity** | ⚠️ Time sync, clock drift handling | ✅ Simpler (server handles time) |
| **Security** | ✅ Secrets never leave device | ⚠️ Secrets transmitted to server |

### Decision Made

**Offline-First Architecture**

### Rationale

1. **100% Availability**: Codes must work in airplane mode, underground, or during network outages
2. **Zero Server Dependency**: Code generation doesn't require server communication
3. **Privacy**: Secrets never transmitted to server (zero-knowledge architecture)
4. **Scalability**: 1 billion users generating codes doesn't require server infrastructure
5. **Low Latency**: Local computation (<10ms) vs network round-trip (100-500ms)

**Why NOT Online-First:**

- **Availability Risk**: Network outages prevent code generation (unacceptable for MFA)
- **Server Load**: 3 billion code generations/day would require massive infrastructure
- **Privacy Concerns**: Secrets transmitted to server (even if encrypted)
- **Latency**: Network round-trip adds 100-500ms delay

### Implementation Details

**TOTP Algorithm (RFC 6238):**

```
Code Generation (Offline):
  1. Get current Unix timestamp
  2. Calculate time step: timestamp / 30
  3. HMAC-SHA1(secret, time_step) → 20-byte hash
  4. Dynamic truncation → 6-digit code
  5. Display code (valid for 30 seconds)

No network calls required!
```

**Time Synchronization:**

- NTP sync when online (calculate offset)
- Store offset for offline use
- Server accepts ±1 time step (90-second window)

### Trade-offs Accepted

- **Time Sync Complexity**: Must handle clock drift (NTP sync, offset storage)
- **No Real-Time Updates**: Account metadata syncs when online (eventual consistency)
- **User Responsibility**: User must ensure device time is accurate

### When to Reconsider

- If real-time account management is required (rare)
- If server-side code generation is mandated by compliance (unlikely)

---

## 2. TOTP vs HOTP vs SMS OTP

### The Problem

Choose the authentication method for generating one-time passwords.

### Options Considered

| Feature | TOTP (Time-Based) | HOTP (Counter-Based) | SMS OTP |
|---------|-------------------|----------------------|---------|
| **Offline Support** | ✅ Yes (uses device time) | ✅ Yes (uses counter) | ❌ No (requires SMS) |
| **Synchronization** | ⚠️ Time sync needed | ✅ Counter sync (simpler) | ✅ Server sends SMS |
| **Replay Protection** | ✅ Time window (30s) | ⚠️ Counter must increment | ✅ One-time use |
| **User Experience** | ✅ Auto-refresh every 30s | ⚠️ Manual refresh | ✅ Push notification |
| **Security** | ✅ Strong (HMAC-SHA1) | ✅ Strong (HMAC-SHA1) | ⚠️ SMS interception risk |
| **Cost** | ✅ Free (no SMS fees) | ✅ Free | ❌ SMS costs ($0.01-0.05 each) |
| **Scalability** | ✅ Infinite (client-side) | ✅ Infinite | ❌ Limited by SMS provider |
| **Industry Standard** | ✅ RFC 6238 (widely adopted) | ⚠️ RFC 4226 (less common) | ⚠️ Proprietary |

### Decision Made

**TOTP (Time-Based One-Time Password)**

### Rationale

1. **Industry Standard**: RFC 6238 is widely adopted (Google Authenticator, Microsoft Authenticator, Authy)
2. **Offline Support**: Works without internet (uses device time)
3. **Auto-Refresh**: Codes automatically update every 30 seconds (better UX)
4. **Replay Protection**: Time window prevents replay attacks
5. **No SMS Costs**: Free to use (no per-code charges)
6. **Scalability**: Client-side generation (no server load)

**Why NOT HOTP:**

- **Manual Refresh**: User must manually refresh codes (worse UX)
- **Counter Sync Issues**: Counter can desync if codes generated but not used
- **Less Common**: Fewer services support HOTP

**Why NOT SMS OTP:**

- **No Offline Support**: Requires SMS network (doesn't work in airplane mode)
- **Security Risk**: SMS interception (SIM swapping, SS7 attacks)
- **Cost**: SMS fees add up ($0.01-0.05 per code)
- **Scalability**: Limited by SMS provider throughput
- **Dependency**: Requires SMS gateway infrastructure

### Implementation Details

**TOTP Algorithm:**

```
Time Step Calculation:
  time_step = floor(current_timestamp / 30)

Code Generation:
  hmac = HMAC-SHA1(secret, time_step)
  code = dynamic_truncation(hmac) % 1000000
  format: 6-digit code (e.g., "847362")
```

**Time Window Tolerance:**

- Server accepts ±1 time step (90-second window total)
- Handles clock drift up to 30 seconds

### Trade-offs Accepted

- **Time Sync Required**: Must handle clock drift (NTP sync, offset storage)
- **30-Second Window**: Codes expire quickly (security vs UX trade-off)

### When to Reconsider

- If HOTP counter-based approach is required by compliance
- If SMS OTP is mandated (though less secure)

---

## 3. Device HSM vs Software Encryption

### The Problem

How to encrypt secrets at rest on the device - use hardware security module (HSM) or software-only encryption?

### Options Considered

| Feature | Device HSM (Keychain/Keystore) | Software Encryption |
|---------|-------------------------------|---------------------|
| **Hardware Backing** | ✅ Secure Enclave / TEE | ❌ Software-only |
| **Tamper Resistance** | ✅ Hardware-protected | ❌ Vulnerable to memory dumps |
| **Biometric Integration** | ✅ Native (Face ID / fingerprint) | ⚠️ App-level implementation |
| **Performance** | ✅ Fast (hardware-accelerated) | ⚠️ Slower (CPU-based) |
| **Platform Support** | ✅ iOS Keychain, Android Keystore | ✅ Cross-platform |
| **Key Extraction** | ✅ Keys never leave HSM | ❌ Keys in app memory |
| **Root/Jailbreak Protection** | ✅ Hardware isolation | ❌ Vulnerable if device compromised |

### Decision Made

**Device HSM (iOS Keychain / Android Keystore)**

### Rationale

1. **Hardware Security**: Secure Enclave (iOS) and TEE (Android) provide hardware-backed encryption
2. **Tamper Resistance**: Keys never leave hardware chip (even if device is jailbroken)
3. **Biometric Integration**: Native Face ID / fingerprint unlock (better UX)
4. **Industry Standard**: iOS Keychain and Android Keystore are platform-recommended
5. **Root/Jailbreak Protection**: Hardware isolation protects keys even if OS is compromised

**Why NOT Software Encryption:**

- **Memory Vulnerabilities**: Keys in app memory can be extracted via memory dumps
- **Root/Jailbreak Risk**: If device is compromised, software encryption is vulnerable
- **No Hardware Backing**: Lacks tamper-resistant hardware protection
- **Biometric Complexity**: Must implement biometric unlock at app level

### Implementation Details

**iOS Keychain:**

```
Keychain Access:
  - Uses Secure Enclave (hardware chip)
  - Keys stored in encrypted keychain
  - Biometric unlock: Face ID / Touch ID
  - App-specific access control
  - Keys never exposed to app code
```

**Android Keystore:**

```
Keystore Access:
  - Hardware-backed Keystore (if available)
  - Software Keystore (fallback)
  - Biometric unlock: fingerprint / face
  - App-specific access control
  - Keys protected by TEE (Trusted Execution Environment)
```

**Encryption Flow:**

```
1. App requests master key from HSM
2. HSM prompts for biometric (Face ID / fingerprint)
3. HSM returns temporary key (never stored in app)
4. App encrypts/decrypts secrets using temporary key
5. Temporary key cleared from memory
```

### Trade-offs Accepted

- **Platform Dependency**: Different APIs for iOS vs Android
- **Hardware Requirement**: Requires Secure Enclave / TEE (not available on all devices)
- **Fallback Needed**: Software encryption fallback for older devices

### When to Reconsider

- If cross-platform software encryption is required (less secure)
- If hardware HSM is unavailable (use software encryption as fallback)

---

## 4. Cloud Backup vs Local-Only Storage

### The Problem

Should accounts be backed up to cloud for recovery, or stored only locally?

### Options Considered

| Feature | Cloud Backup | Local-Only Storage |
|---------|--------------|-------------------|
| **Account Recovery** | ✅ Restore if device lost | ❌ Accounts lost if device lost |
| **Multi-Device Sync** | ✅ Same accounts on all devices | ❌ Must add accounts on each device |
| **Privacy** | ⚠️ Encrypted (zero-knowledge) | ✅ No cloud storage |
| **Password Recovery** | ❌ No password reset (by design) | N/A |
| **Server Infrastructure** | ⚠️ Requires backup service | ✅ No server needed |
| **User Control** | ✅ User controls encryption key | ✅ Full local control |
| **Compliance** | ⚠️ Data stored in cloud | ✅ No cloud data |

### Decision Made

**Cloud Backup with Zero-Knowledge Encryption (Optional)**

### Rationale

1. **Account Recovery**: Users can restore accounts if device is lost/stolen
2. **Multi-Device Support**: Same accounts on phone, tablet, watch
3. **Zero-Knowledge**: Server never sees plaintext secrets (encrypted with user password)
4. **User Control**: User controls encryption key (password)
5. **Optional**: Users can disable backup if they prefer local-only

**Why NOT Local-Only:**

- **Account Loss Risk**: If device is lost, all accounts are lost (no recovery)
- **Multi-Device Friction**: Must manually add accounts on each device
- **User Experience**: Poor UX for users with multiple devices

**Why Zero-Knowledge Encryption:**

- **Privacy**: Server cannot decrypt accounts (even if compromised)
- **User Control**: User owns encryption key (password)
- **No Password Reset**: By design - if password lost, backup cannot be decrypted

### Implementation Details

**Backup Encryption:**

```
1. User enters backup password
2. PBKDF2(password, salt, 100k iterations) → cloud encryption key
3. Decrypt secrets with HSM key
4. Re-encrypt with cloud key
5. Upload encrypted accounts to server
6. Server stores encrypted data (never sees plaintext)
```

**Restore Process:**

```
1. User enters backup password
2. Derive same cloud key (PBKDF2)
3. Download encrypted accounts from server
4. Decrypt with cloud key
5. Re-encrypt with new device's HSM key
6. Store in new device's local database
```

**Security Properties:**

- Server never sees plaintext secrets
- User controls encryption key (password)
- No password reset (security feature)
- Each device has its own HSM master key

### Trade-offs Accepted

- **Server Infrastructure**: Requires backup service (minimal cost)
- **No Password Reset**: Lost password = lost backup (by design)
- **Cloud Storage**: Encrypted data stored in cloud (zero-knowledge)

### When to Reconsider

- If compliance requires local-only storage (no cloud)
- If users prefer local-only (make backup optional)

---

## 5. Push Notifications vs TOTP-Only

### The Problem

Should the app support push notification approvals (Microsoft Authenticator style) or only TOTP codes?

### Options Considered

| Feature | Push Notifications | TOTP-Only |
|---------|-------------------|-----------|
| **User Experience** | ✅ One-tap approval | ⚠️ Must type 6-digit code |
| **Convenience** | ✅ Faster (2-5 seconds) | ⚠️ Slower (10-20 seconds) |
| **Offline Support** | ❌ Requires internet | ✅ Works offline |
| **Server Infrastructure** | ⚠️ Requires push service | ✅ No server needed |
| **Fallback** | ✅ TOTP codes always available | N/A |
| **Security** | ✅ Device verification, timeout | ✅ Strong (HMAC-SHA1) |
| **Complexity** | ⚠️ Push service, WebSocket, device registration | ✅ Simple (TOTP only) |
| **Platform Support** | ⚠️ FCM (Android), APNs (iOS) | ✅ Universal |

### Decision Made

**Push Notifications with TOTP Fallback**

### Rationale

1. **Better UX**: One-tap approval vs typing 6-digit code
2. **Faster**: 2-5 seconds vs 10-20 seconds
3. **Fallback Available**: TOTP codes always work if push fails
4. **Industry Standard**: Microsoft Authenticator, Google Smart Lock use push
5. **Optional Feature**: Users can disable push, use TOTP only

**Why NOT TOTP-Only:**

- **Worse UX**: Typing 6-digit codes is tedious
- **Slower**: 10-20 seconds vs 2-5 seconds
- **Competitive Disadvantage**: Most modern authenticators support push

**Why Push with TOTP Fallback:**

- **Best of Both**: Convenience of push, reliability of TOTP
- **Offline Support**: TOTP works when push unavailable
- **User Choice**: Users can prefer TOTP if they want

### Implementation Details

**Push Approval Flow:**

```
1. User attempts login on website
2. Website sends approval request to authenticator service
3. Service sends push notification to user's device
4. User taps "Approve" or "Deny"
5. App sends response to service
6. Service notifies website via WebSocket
7. Website completes or rejects authentication
```

**Security Features:**

- 30-second timeout (prevents replay attacks)
- Device verification (only registered devices)
- Rate limiting (max 5 requests per 15 minutes)
- Biometric confirmation (optional, for sensitive accounts)

**Fallback:**

- If push fails or user prefers TOTP, use TOTP codes
- TOTP always available (offline support)

### Trade-offs Accepted

- **Server Infrastructure**: Requires push service (FCM/APNs), WebSocket service
- **Internet Dependency**: Push requires internet (TOTP fallback works offline)
- **Complexity**: More complex than TOTP-only (device registration, push handling)

### When to Reconsider

- If push infrastructure is unavailable (use TOTP-only)
- If users prefer TOTP-only (make push optional)

---

## 6. Synchronous vs Asynchronous Sync

### The Problem

Should account metadata synchronization happen synchronously (blocking) or asynchronously (background)?

### Options Considered

| Feature | Asynchronous (Background) | Synchronous (Blocking) |
|---------|--------------------------|------------------------|
| **User Experience** | ✅ Non-blocking (codes work immediately) | ❌ Blocks until sync completes |
| **Offline Support** | ✅ Codes work offline, sync when online | ❌ Requires internet for sync |
| **Latency** | ✅ Codes displayed immediately | ❌ Wait for sync (100-500ms) |
| **Error Handling** | ✅ Sync failures don't affect codes | ❌ Sync failures block code display |
| **Complexity** | ⚠️ Background service, state management | ✅ Simpler (blocking call) |
| **Battery Impact** | ⚠️ Background sync consumes battery | ✅ No background processes |

### Decision Made

**Asynchronous (Background) Sync**

### Rationale

1. **Offline-First**: Codes must work immediately, even if sync fails
2. **Non-Blocking UX**: Users see codes instantly (no wait for sync)
3. **Resilience**: Sync failures don't prevent code generation
4. **Efficiency**: Sync only when online (periodic, not on every code generation)

**Why NOT Synchronous:**

- **Blocks Code Display**: Users wait for sync before seeing codes (bad UX)
- **Offline Failure**: Sync fails offline, codes don't display (unacceptable)
- **Latency**: Adds 100-500ms delay to code display

### Implementation Details

**Background Sync Service:**

```
Sync Triggers:
  - Every 6 hours (periodic)
  - On app open (if last sync > 1 hour ago)
  - Manual refresh (user-initiated)

Sync Process:
  1. Check internet connectivity
  2. Send account IDs and last sync timestamps
  3. Server returns updates, deleted, new accounts
  4. Update local database (non-blocking)
  5. Update sync timestamps
```

**Code Generation (Non-Blocking):**

```
1. User opens app
2. Codes generated immediately (offline)
3. Background sync triggers (if online)
4. Metadata updates applied when sync completes
5. Codes continue to work during sync
```

### Trade-offs Accepted

- **Eventual Consistency**: Metadata may be stale if offline (acceptable)
- **Background Complexity**: Requires background service, state management
- **Battery Impact**: Background sync consumes battery (minimal, periodic)

### When to Reconsider

- If real-time metadata is critical (rare for authenticator apps)
- If battery impact is unacceptable (reduce sync frequency)

---

## 7. SQLite vs NoSQL for Local Storage

### The Problem

What database should be used for local storage on mobile devices?

### Options Considered

| Feature | SQLite | Realm | NoSQL (JSON Files) |
|---------|--------|-------|-------------------|
| **Platform Support** | ✅ iOS, Android, universal | ⚠️ Realm-specific | ✅ Universal |
| **Performance** | ✅ Fast (optimized for mobile) | ✅ Fast (object database) | ⚠️ Slower (file I/O) |
| **Query Flexibility** | ✅ SQL queries, indexes | ✅ Object queries | ❌ Manual parsing |
| **ACID Compliance** | ✅ Full ACID | ✅ ACID | ❌ No transactions |
| **Encryption** | ✅ SQLCipher (encrypted SQLite) | ✅ Built-in encryption | ⚠️ Manual encryption |
| **Size** | ✅ Small (~500KB) | ⚠️ Larger (~5MB) | ✅ Minimal |
| **Maturity** | ✅ Battle-tested (decades) | ⚠️ Newer | ✅ Simple |
| **Migration** | ✅ Schema migrations | ⚠️ Realm migrations | ✅ No schema |

### Decision Made

**SQLite with SQLCipher (Encrypted)**

### Rationale

1. **Platform Standard**: SQLite is built into iOS and Android
2. **Performance**: Optimized for mobile (fast queries, indexes)
3. **ACID Compliance**: Transactions ensure data integrity
4. **Encryption**: SQLCipher provides transparent encryption
5. **Maturity**: Battle-tested for decades (reliable)
6. **Query Flexibility**: SQL queries for complex operations
7. **Small Size**: ~500KB (minimal app size impact)

**Why NOT Realm:**

- **Larger Size**: ~5MB (10x larger than SQLite)
- **Platform Dependency**: Realm-specific (less universal)
- **Newer Technology**: Less battle-tested than SQLite

**Why NOT NoSQL (JSON Files):**

- **No ACID**: No transactions (data integrity risk)
- **Slower**: File I/O is slower than database queries
- **Manual Encryption**: Must implement encryption manually
- **No Query Flexibility**: Must parse JSON manually

### Implementation Details

**SQLite Schema:**

```
CREATE TABLE accounts (
    account_id TEXT PRIMARY KEY,
    issuer TEXT NOT NULL,
    account_name TEXT NOT NULL,
    encrypted_secret TEXT NOT NULL,
    algorithm TEXT DEFAULT 'SHA1',
    digits INTEGER DEFAULT 6,
    period INTEGER DEFAULT 30,
    icon_url TEXT,
    added_at INTEGER NOT NULL,
    last_used INTEGER,
    last_sync INTEGER
);

CREATE INDEX idx_issuer_name ON accounts(issuer, account_name);
```

**SQLCipher Encryption:**

```
// Enable encryption
PRAGMA key = 'master_key_from_hsm';

// All data encrypted at rest
// Transparent to application code
```

### Trade-offs Accepted

- **Schema Migrations**: Must handle schema changes (migration scripts)
- **SQL Knowledge**: Requires SQL knowledge (but simple queries)

### When to Reconsider

- If Realm's object database model is preferred (larger app size)
- If JSON files are sufficient (simpler, but less performant)

---

## 8. PBKDF2 vs Argon2 for Key Derivation

### The Problem

What key derivation function (KDF) should be used for deriving cloud backup encryption keys from user passwords?

### Options Considered

| Feature | PBKDF2 | Argon2 | bcrypt |
|---------|--------|--------|--------|
| **Industry Standard** | ✅ NIST recommended | ✅ Winner of PHC (2015) | ⚠️ Older standard |
| **Memory Hardness** | ❌ CPU-only | ✅ Memory-hard (resistant to ASIC) | ⚠️ Some memory hardness |
| **Platform Support** | ✅ Universal | ⚠️ Requires library | ✅ Universal |
| **Performance** | ✅ Fast (configurable) | ⚠️ Slower (memory-intensive) | ⚠️ Slower |
| **Mobile Optimization** | ✅ Fast on mobile | ⚠️ Memory-intensive (battery drain) | ⚠️ Slower |
| **Iterations** | ✅ 100k iterations (standard) | ✅ Configurable (time, memory) | ✅ Configurable |
| **Security** | ✅ Strong (with high iterations) | ✅ Strong (memory-hard) | ✅ Strong |

### Decision Made

**PBKDF2 with 100,000 Iterations**

### Rationale

1. **Industry Standard**: NIST recommended, widely adopted
2. **Mobile Performance**: Fast on mobile devices (low battery impact)
3. **Platform Support**: Built into iOS and Android (no external library)
4. **Proven Security**: 100k iterations provides strong security
5. **Configurable**: Can increase iterations if needed

**Why NOT Argon2:**

- **Memory-Intensive**: High memory usage (battery drain on mobile)
- **Library Dependency**: Requires external library (larger app size)
- **Mobile Optimization**: Designed for servers, not mobile devices

**Why NOT bcrypt:**

- **Older Standard**: PBKDF2 is more modern and NIST-recommended
- **Performance**: Slower than PBKDF2 on mobile

### Implementation Details

**PBKDF2 Key Derivation:**

```
Cloud Key Derivation:
  cloud_key = PBKDF2(
    password = user_backup_password,
    salt = random_32_bytes,
    iterations = 100000,
    key_length = 32,
    hash_algorithm = SHA256
  )

Time: ~500ms on modern mobile device
Security: Strong (100k iterations)
```

**Security Properties:**

- 100k iterations (industry standard)
- Random salt (prevents rainbow table attacks)
- 32-byte key (AES-256)
- SHA-256 hash algorithm

### Trade-offs Accepted

- **ASIC Vulnerability**: PBKDF2 is CPU-only (vulnerable to ASIC attacks, but acceptable for mobile)
- **Not Memory-Hard**: Argon2 is more resistant to ASIC, but too memory-intensive for mobile

### When to Reconsider

- If Argon2 becomes standard on mobile platforms (currently not)
- If higher security is required (increase PBKDF2 iterations to 200k+)

---

## Summary

| Decision | Choice | Key Rationale |
|----------|--------|---------------|
| **Architecture** | Offline-First | 100% availability, zero server dependency |
| **OTP Method** | TOTP | Industry standard, offline support, auto-refresh |
| **Encryption** | Device HSM | Hardware security, tamper resistance |
| **Backup** | Cloud (Zero-Knowledge) | Account recovery, multi-device, user-controlled |
| **Push Notifications** | Optional (with TOTP fallback) | Better UX, TOTP always available |
| **Sync** | Asynchronous | Non-blocking, offline-first |
| **Local Storage** | SQLite (Encrypted) | Platform standard, ACID, performance |
| **Key Derivation** | PBKDF2 (100k iterations) | Industry standard, mobile-optimized |

**Design Philosophy**: **Security and availability over convenience** - offline operation, zero-knowledge encryption, and hardware security prioritize security and reliability, even if it means more complexity and no password reset.

