# 3.5.7 Design Authenticator App (Microsoft Authenticator / Google Authenticator)

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

Design an **authenticator app** like Microsoft Authenticator or Google Authenticator that generates time-based one-time passwords (TOTP) for multi-factor authentication (MFA). The system must work **completely offline** (no internet required for code generation), support **background synchronization** when online, enable **multi-device access** with secure key sharing, and provide **backup/recovery** mechanisms without compromising security.

**Key Challenges:**

- **Offline Operation**: Generate TOTP codes without internet connectivity (airplane mode, no signal)
- **Time Synchronization**: Handle clock drift between device and server (TOTP requires accurate time)
- **Secure Key Storage**: Encrypt secrets at rest using device hardware security modules (HSM)
- **Multi-Device Sync**: Share accounts across devices securely (phone, tablet, watch)
- **Background Sync**: Update account metadata, icons, names when online without user interaction
- **Backup and Recovery**: Allow account recovery if device is lost without exposing secrets
- **Push Notifications**: Support push-based approval requests (Microsoft Authenticator style)
- **Account Management**: Add, remove, organize accounts with proper encryption

**Real-World Examples:**

- **Google Authenticator**: 500M+ users, offline TOTP generation, QR code setup
- **Microsoft Authenticator**: 200M+ users, push notifications, cloud backup
- **Authy**: Multi-device sync, encrypted cloud backup, account recovery
- **1Password Authenticator**: Integrated with password manager, secure vault

**The Core Challenge:**

Traditional authentication requires server communication, but authenticator apps must:

1. **Work offline**: Generate codes in airplane mode (no network)
2. **Stay synchronized**: Handle time drift, account updates, multi-device sync
3. **Maintain security**: Never expose secrets, even during backup/recovery
4. **Scale globally**: Support millions of users with minimal server infrastructure

---

## 2. Requirements and Scale Estimation

### Functional Requirements (FRs)

1. **TOTP Code Generation**: Generate 6-digit codes that change every 30 seconds using RFC 6238
2. **Offline Operation**: Generate codes without internet connectivity
3. **QR Code Setup**: Scan QR codes to add accounts (contains secret key + metadata)
4. **Manual Entry**: Support manual secret key entry (Base32 encoded)
5. **Multi-Device Sync**: Share accounts across multiple devices securely
6. **Background Sync**: Update account names, icons, metadata when online
7. **Cloud Backup**: Encrypted backup to cloud for account recovery
8. **Push Notifications**: Receive and approve/deny authentication requests (Microsoft style)
9. **Account Organization**: Group accounts by service, search, favorites
10. **Time Synchronization**: Detect and handle clock drift between device and server

### Non-Functional Requirements (NFRs)

1. **Offline-First**: Must work 100% offline for code generation
2. **Low Latency**: Code generation < 10ms (local computation)
3. **Security**: Secrets encrypted at rest using device HSM (Hardware Security Module)
4. **Availability**: 99.9% uptime for sync services (code generation doesn't need servers)
5. **Privacy**: Zero-knowledge architecture (server never sees plaintext secrets)
6. **Scalability**: Support 1B+ users with minimal server load (most operations are client-side)

### Scale Estimation

| Metric | Assumption | Calculation | Result |
|--------|------------|-------------|--------|
| **Total Users** | Global adoption | - | 1 billion users |
| **Active Users** | Daily usage | 30% of total | 300M daily active users |
| **Code Generations** | Per user per day | 300M × 10 codes/day | 3 billion codes/day |
| **Code Generation QPS** | Peak load | $\frac{3 \text{B}}{86400 \text{s}}$ | ~35k QPS (client-side, no server load) |
| **Sync Operations** | Account updates | 300M × 0.1 syncs/day | 30M syncs/day |
| **Sync QPS** | Peak load | $\frac{30 \text{M}}{86400 \text{s}}$ | ~350 QPS (server-side) |
| **Push Notifications** | Approval requests | 300M × 2 requests/day | 600M requests/day |
| **Push QPS** | Peak load | $\frac{600 \text{M}}{86400 \text{s}}$ | ~7k QPS |
| **Storage per User** | Accounts + metadata | 10 accounts × 2 KB | 20 KB/user |
| **Total Storage** | 1B users | $1 \text{B} \times 20 \text{KB}$ | 20 TB (encrypted backups) |
| **Bandwidth** | Sync operations | 350 QPS × 5 KB/request | ~1.75 MB/s |

**Key Insight**: Most operations (code generation) are **client-side** and require **zero server infrastructure**. Only sync, backup, and push notifications need servers.

---

## 3. High-Level Architecture

> 📊 **See detailed architecture:** [High-Level Design Diagrams](./hld-diagram.md)

The architecture follows an **offline-first** design where code generation happens entirely on-device, with optional cloud services for sync, backup, and push notifications.

### Core Components

```
┌─────────────────────────────────────────────────────────────────┐
│                    Mobile Device (iOS/Android)                   │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │              Authenticator App (Client)                  │  │
│  │                                                            │  │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │  │
│  │  │ TOTP Engine  │  │ Key Storage  │  │ UI Layer     │  │  │
│  │  │ (Offline)    │  │ (Encrypted)  │  │              │  │  │
│  │  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  │  │
│  │         │                 │                  │           │  │
│  │         └─────────────────┼──────────────────┘           │  │
│  │                           │                              │  │
│  │  ┌────────────────────────┴──────────────────────────┐  │  │
│  │  │         Device HSM (Hardware Security Module)      │  │  │
│  │  │         - Keychain (iOS) / Keystore (Android)     │  │  │
│  │  │         - Hardware-backed encryption               │  │  │
│  │  └────────────────────────────────────────────────────┘  │  │
│  └───────────────────────────┬──────────────────────────────┘  │
│                                │                                 │
│                                │ (Optional: When Online)         │
└────────────────────────────────┼─────────────────────────────────┘
                                 │
                    ┌────────────┴────────────┐
                    │                         │
                    ▼                         ▼
        ┌─────────────────────┐   ┌─────────────────────┐
        │  Sync Service       │   │  Push Notification  │
        │  (Account Updates)   │   │  Service           │
        │                      │   │  (Approval Requests)│
        │  - Metadata sync     │   │                     │
        │  - Multi-device      │   │  - FCM (Android)    │
        │  - Cloud backup      │   │  - APNs (iOS)       │
        └──────────┬──────────┘   └──────────┬──────────┘
                   │                         │
                   └─────────────┬───────────┘
                                 │
                                 ▼
                    ┌─────────────────────────┐
                    │   Database (Encrypted)  │
                    │                         │
                    │  - Account metadata     │
                    │  - Encrypted secrets    │
                    │  - Device registrations │
                    │  - Push tokens          │
                    └─────────────────────────┘
```

### Key Design Principles

1. **Offline-First**: TOTP generation requires **zero network calls** (works in airplane mode)
2. **Zero-Knowledge**: Server never sees plaintext secrets (encrypted client-side)
3. **Hardware Security**: Use device HSM for key encryption (Keychain/Keystore)
4. **Eventual Consistency**: Account metadata syncs when online (codes work offline)
5. **Multi-Device**: Encrypted key sharing via cloud (user controls encryption key)

---

## 4. Detailed Component Design

### 4.1 TOTP Code Generation (Offline)

**The Core Algorithm**: TOTP (Time-based One-Time Password) from RFC 6238

**How It Works:**

1. **Shared Secret**: User and server both know a secret key (established during setup)
2. **Time Step**: Current time divided by 30 seconds → counter value
3. **HMAC-SHA1**: Compute HMAC-SHA1(secret, counter) → 20-byte hash
4. **Dynamic Truncation**: Extract 31-bit value from hash
5. **6-Digit Code**: Modulo 1,000,000 → 6-digit code (000000-999999)

**Why It Works Offline:**

- **Deterministic**: Same secret + same time = same code
- **No Server Needed**: Device computes code locally using current time
- **Time Window**: Server accepts codes from ±1 time step (90-second window)

**Implementation Flow:**

```
1. User opens app → TOTP Engine
2. TOTP Engine:
   a. Get current Unix timestamp: time()
   b. Calculate time step: timestamp // 30
   c. Retrieve encrypted secret from Key Storage
   d. Decrypt secret using HSM (hardware-backed)
   e. Compute: HMAC-SHA1(secret, time_step)
   f. Extract 6-digit code via dynamic truncation
   g. Display code to user
3. Code valid for 30 seconds, then regenerates
```

**Time Synchronization:**

- **Problem**: Device clock drift causes code mismatch
- **Solution**: 
  - NTP sync on device (automatic)
  - Server accepts ±1 time step (90-second window)
  - Manual time correction if drift > 30 seconds

*See pseudocode.md::generate_totp() for implementation*

### 4.2 Secure Key Storage

**The Problem**: Secrets must be encrypted at rest, but code generation needs fast decryption.

**Solution**: Device Hardware Security Module (HSM)

**iOS (Keychain):**
- Uses Secure Enclave (hardware chip)
- Keys never leave device
- Biometric unlock (Face ID / Touch ID)
- App-specific access control

**Android (Keystore):**
- Hardware-backed Keystore (if available)
- Software Keystore (fallback)
- Biometric unlock (fingerprint / face)
- App-specific access control

**Storage Structure:**

```
Account Record:
{
  "account_id": "uuid-123",
  "issuer": "Google",
  "account_name": "user@example.com",
  "encrypted_secret": "AES-256-GCM(secret_key, master_key)",
  "algorithm": "SHA1",  // or SHA256, SHA512
  "digits": 6,          // code length
  "period": 30,         // time step (seconds)
  "icon_url": "https://...",
  "added_at": "2024-01-01T00:00:00Z",
  "last_used": "2024-01-15T10:30:00Z"
}

Master Key:
- Stored in device HSM (Keychain/Keystore)
- Never exposed to app code
- Unlocked via biometrics
- Used to encrypt/decrypt account secrets
```

**Encryption Flow:**

```
1. User adds account → Secret key received
2. App requests master key from HSM (biometric prompt)
3. HSM returns master key (temporary, in memory)
4. App encrypts secret: AES-256-GCM(secret, master_key)
5. Store encrypted_secret in app database
6. Master key cleared from memory
7. When generating code:
   a. Request master key from HSM (biometric prompt)
   b. Decrypt secret using master key
   c. Generate TOTP code
   d. Master key cleared from memory
```

*See pseudocode.md::encrypt_secret() and pseudocode.md::decrypt_secret() for implementation*

### 4.3 Account Setup (QR Code / Manual Entry)

**Method 1: QR Code Scanning**

**QR Code Format**: `otpauth://totp/{issuer}:{account}?secret={secret}&issuer={issuer}&algorithm={algorithm}&digits={digits}&period={period}`

**Example:**
```
otpauth://totp/Google:user@example.com?secret=JBSWY3DPEHPK3PXP&issuer=Google&algorithm=SHA1&digits=6&period=30
```

**Flow:**

```
1. Server generates QR code with secret + metadata
2. User opens authenticator app → "Add Account" → "Scan QR Code"
3. App scans QR code → parses otpauth:// URL
4. Extract: secret, issuer, account_name, algorithm, digits, period
5. Validate secret (Base32 format, correct length)
6. Encrypt secret using HSM master key
7. Store account record in local database
8. Generate first TOTP code to verify
9. User enters code on server → server validates → account linked
```

**Method 2: Manual Entry**

**Flow:**

```
1. Server displays secret as Base32 string: "JBSWY3DPEHPK3PXP"
2. User opens app → "Add Account" → "Enter Setup Key"
3. User types: issuer, account_name, secret
4. App validates secret (Base32, correct length)
5. Encrypt and store (same as QR code flow)
```

*See pseudocode.md::parse_otpauth_url() and pseudocode.md::add_account() for implementation*

### 4.4 Background Synchronization

**The Problem**: Account metadata (name, icon, issuer) may change on server, but app works offline.

**Solution**: Background sync when online (iOS Background App Refresh, Android WorkManager)

**What Syncs:**

- **Account metadata**: Name, issuer, icon URL
- **Account status**: Disabled, deleted, renamed
- **New accounts**: Added on another device
- **Account order**: User preferences (favorites, groups)

**What Doesn't Sync:**

- **Secrets**: Never sent to server (zero-knowledge)
- **TOTP codes**: Generated locally, never synced

**Sync Flow:**

```
1. App detects internet connectivity
2. Background sync service triggers (every 6 hours, or on app open)
3. App sends to server:
   {
     "device_id": "uuid-device-123",
     "accounts": [
       {"account_id": "uuid-1", "last_sync": "2024-01-15T10:00:00Z"},
       {"account_id": "uuid-2", "last_sync": "2024-01-15T09:00:00Z"}
     ]
   }
4. Server responds:
   {
     "updates": [
       {
         "account_id": "uuid-1",
         "account_name": "Updated Name",  // Changed
         "icon_url": "https://new-icon.png"  // Changed
       }
     ],
     "deleted": ["uuid-3"],  // Account deleted on another device
     "new_accounts": []  // None added on other devices
   }
5. App updates local database:
   - Update account metadata
   - Remove deleted accounts
   - Add new accounts (if any, requires user approval)
6. Sync timestamp updated
```

**Privacy**: Server only sees account IDs and sync timestamps, never secrets.

*See pseudocode.md::sync_accounts() for implementation*

### 4.5 Multi-Device Synchronization

**The Problem**: User wants same accounts on phone, tablet, and watch.

**Solution**: Encrypted cloud backup with user-controlled encryption key.

**Architecture:**

```
Device 1 (Phone)                    Device 2 (Tablet)
     │                                    │
     │ 1. User adds account               │
     │    Secret encrypted locally        │
     │                                    │
     │ 2. Encrypt account with            │
     │    user's cloud key                │
     │                                    │
     ▼                                    ▼
┌─────────────────┐              ┌─────────────────┐
│  Cloud Backup   │              │  Cloud Backup   │
│  (Encrypted)    │◄─────────────┤  (Encrypted)    │
│                 │   Sync       │                 │
└─────────────────┘              └─────────────────┘
```

**Encryption Model:**

1. **User Cloud Key**: Derived from user password (PBKDF2, 100k iterations)
2. **Account Encryption**: Each account encrypted with cloud key before upload
3. **Zero-Knowledge**: Server never sees plaintext secrets

**Flow:**

```
Device 1 (Add Account):
1. User adds account → Secret encrypted with device HSM
2. User enables cloud backup → Enter cloud password
3. Derive cloud key: PBKDF2(password, salt, 100k iterations)
4. Encrypt account record: AES-256-GCM(account, cloud_key)
5. Upload encrypted account to cloud
6. Server stores: {encrypted_account, device_id, timestamp}

Device 2 (Restore):
1. User installs app on Device 2
2. User enables cloud backup → Enter cloud password
3. Derive cloud key: PBKDF2(password, salt, 100k iterations)
4. Download encrypted accounts from cloud
5. Decrypt each account: AES-256-GCM(encrypted_account, cloud_key)
6. Re-encrypt with Device 2's HSM master key
7. Store in Device 2's local database
8. Accounts now available on Device 2
```

**Security Properties:**

- **Server Blind**: Server never sees plaintext secrets
- **Password Required**: User must remember cloud password (no password reset)
- **Device Independence**: Each device has its own HSM master key
- **Revocation**: User can delete cloud backup (all devices lose sync)

*See pseudocode.md::backup_to_cloud() and pseudocode.md::restore_from_cloud() for implementation*

### 4.6 Push Notification Approval (Microsoft Authenticator Style)

**The Problem**: Typing 6-digit codes is tedious. Can we approve requests with a tap?

**Solution**: Push notifications with approve/deny buttons.

**Flow:**

```
1. User attempts login on website
2. Website sends push request to authenticator service
3. Authentator service:
   a. Validates user identity
   b. Sends push notification to user's registered devices
   c. Waits for approval (30-second timeout)
4. User receives push notification on phone
5. User taps "Approve" or "Deny"
6. App sends approval/denial to authenticator service
7. Authentator service notifies website
8. Website completes login or rejects
```

**Architecture:**

```
Website                    Authenticator Service          Mobile App
   │                              │                          │
   │ 1. POST /auth/request        │                          │
   │    {user_id, device_id}     │                          │
   │─────────────────────────────>│                          │
   │                              │                          │
   │                              │ 2. Lookup push token     │
   │                              │    for user's devices    │
   │                              │                          │
   │                              │ 3. Send push via FCM/APNs│
   │                              │─────────────────────────>│
   │                              │                          │
   │                              │                          │ 4. Show notification
   │                              │                          │    "Approve login?"
   │                              │                          │
   │                              │                          │ 5. User taps "Approve"
   │                              │                          │
   │                              │ 6. POST /auth/approve    │
   │                              │<─────────────────────────│
   │                              │    {request_id, approved}│
   │                              │                          │
   │ 7. WebSocket: Login approved │                          │
   │<─────────────────────────────│                          │
   │                              │                          │
```

**Security Considerations:**

- **Request Validation**: Server validates user identity before sending push
- **Time Window**: 30-second timeout (prevents replay attacks)
- **Device Verification**: Only registered devices receive pushes
- **Rate Limiting**: Max 5 push requests per 15 minutes per user

*See pseudocode.md::send_push_approval() and pseudocode.md::handle_approval() for implementation*

### 4.7 Time Synchronization and Clock Drift

**The Problem**: TOTP requires accurate time. Device clock drift causes code mismatches.

**Solutions:**

**1. NTP Synchronization (Automatic)**

- iOS/Android sync time via NTP automatically
- Most devices accurate within ±1 second
- No app code needed

**2. Server Time Window (Tolerance)**

- Server accepts codes from ±1 time step
- 30-second time step → 90-second acceptance window
- Handles minor clock drift automatically

**3. Manual Time Correction**

- If drift > 30 seconds, codes fail
- App detects failure → prompts user to sync time
- User enables "Automatic Date & Time" in device settings

**4. Time Step Validation**

```
Server-side validation:
1. Receive TOTP code from user
2. Get current server time: T_server
3. Calculate expected time step: T_server // 30
4. Generate codes for time steps: [expected - 1, expected, expected + 1]
5. Compare user's code with all three
6. If match → accept (handles ±30 second drift)
```

*See pseudocode.md::validate_totp_with_drift() for implementation*

---

## 5. Data Models

### 5.1 Local Database Schema (SQLite)

```sql
-- Accounts table (encrypted at rest)
CREATE TABLE accounts (
    account_id TEXT PRIMARY KEY,
    issuer TEXT NOT NULL,
    account_name TEXT NOT NULL,
    encrypted_secret TEXT NOT NULL,  -- AES-256-GCM encrypted
    algorithm TEXT DEFAULT 'SHA1',    -- SHA1, SHA256, SHA512
    digits INTEGER DEFAULT 6,
    period INTEGER DEFAULT 30,
    icon_url TEXT,
    icon_data BLOB,                   -- Cached icon
    added_at INTEGER NOT NULL,        -- Unix timestamp
    last_used INTEGER,                -- Unix timestamp
    is_favorite INTEGER DEFAULT 0,
    sort_order INTEGER DEFAULT 0,
    created_at INTEGER NOT NULL,
    updated_at INTEGER NOT NULL
);

-- Device metadata
CREATE TABLE device_metadata (
    device_id TEXT PRIMARY KEY,
    device_name TEXT,
    last_sync INTEGER,
    cloud_backup_enabled INTEGER DEFAULT 0,
    push_token TEXT,
    created_at INTEGER NOT NULL
);

-- Sync state
CREATE TABLE sync_state (
    account_id TEXT PRIMARY KEY,
    last_synced_at INTEGER,
    sync_version INTEGER DEFAULT 0,
    FOREIGN KEY (account_id) REFERENCES accounts(account_id)
);
```

### 5.2 Cloud Backup Schema (Server-Side)

```sql
-- Encrypted account backups (zero-knowledge)
CREATE TABLE account_backups (
    backup_id TEXT PRIMARY KEY,
    user_id TEXT NOT NULL,
    device_id TEXT NOT NULL,
    encrypted_account_data TEXT NOT NULL,  -- AES-256-GCM encrypted
    account_id TEXT NOT NULL,
    created_at INTEGER NOT NULL,
    updated_at INTEGER NOT NULL,
    INDEX idx_user_device (user_id, device_id),
    INDEX idx_account (account_id)
);

-- Device registrations
CREATE TABLE device_registrations (
    device_id TEXT PRIMARY KEY,
    user_id TEXT NOT NULL,
    device_name TEXT,
    push_token TEXT,                  -- FCM/APNs token
    platform TEXT,                    -- ios, android
    last_seen INTEGER,
    created_at INTEGER NOT NULL,
    INDEX idx_user (user_id)
);

-- Push approval requests
CREATE TABLE approval_requests (
    request_id TEXT PRIMARY KEY,
    user_id TEXT NOT NULL,
    device_id TEXT,
    website_domain TEXT,
    status TEXT,                      -- pending, approved, denied, expired
    created_at INTEGER NOT NULL,
    expires_at INTEGER NOT NULL,
    responded_at INTEGER,
    INDEX idx_user_status (user_id, status),
    INDEX idx_expires (expires_at)
);
```

---

## 6. Key Algorithms

### 6.1 TOTP Generation Algorithm

**Input**: Secret key (Base32), current timestamp

**Output**: 6-digit code (000000-999999)

**Steps:**

1. **Decode Secret**: Base32 decode secret → binary key
2. **Calculate Time Step**: `counter = floor(timestamp / 30)`
3. **HMAC-SHA1**: `hmac = HMAC-SHA1(secret, counter)` (20 bytes)
4. **Dynamic Truncation**:
   - `offset = hmac[19] & 0x0F` (last 4 bits)
   - `code = (hmac[offset] & 0x7F) << 24 | (hmac[offset+1] & 0xFF) << 16 | (hmac[offset+2] & 0xFF) << 8 | (hmac[offset+3] & 0xFF)`
   - `code = code % 1000000` (6 digits)
5. **Format**: Pad to 6 digits with leading zeros

**Time Complexity**: O(1) (constant time, ~1ms)

*See pseudocode.md::generate_totp() for detailed implementation*

### 6.2 Secret Encryption (HSM-Based)

**Input**: Plaintext secret, device HSM

**Output**: Encrypted secret (stored in database)

**Steps:**

1. **Request Master Key**: Prompt user for biometric (Face ID / fingerprint)
2. **HSM Unlock**: Device HSM returns master key (temporary, in memory)
3. **Encrypt**: `encrypted = AES-256-GCM(secret, master_key, nonce)`
4. **Store**: Save `encrypted + nonce` in database
5. **Clear Memory**: Master key cleared from memory

**Security**: Master key never leaves device HSM, encrypted secret useless without device.

*See pseudocode.md::encrypt_secret() for implementation*

### 6.3 Cloud Backup Encryption

**Input**: Account data, user password

**Output**: Encrypted backup (uploaded to server)

**Steps:**

1. **Derive Cloud Key**: `cloud_key = PBKDF2(password, salt, 100000 iterations, SHA-256)`
2. **Encrypt Account**: `encrypted = AES-256-GCM(account_json, cloud_key, nonce)`
3. **Upload**: Send `encrypted + nonce + salt` to server
4. **Server Storage**: Server stores encrypted data (never sees plaintext)

**Recovery**: User enters password → derive same cloud key → decrypt accounts.

*See pseudocode.md::backup_to_cloud() for implementation*

---

## 7. Availability and Fault Tolerance

### 7.1 Offline Operation (Primary Use Case)

**Design**: Code generation requires **zero network calls**.

**Fault Scenarios:**

- **No Internet**: Codes still generate (offline-first)
- **Server Down**: Codes still generate (no server dependency)
- **Sync Failure**: Codes still generate (sync is optional)

**Result**: **100% availability** for code generation (device-dependent only).

### 7.2 Sync Service Availability

**Requirement**: 99.9% uptime (sync is optional, codes work offline).

**Strategies:**

1. **Multi-Region Deployment**: Deploy sync service in 3+ regions
2. **Load Balancing**: Distribute requests across regions
3. **Circuit Breaker**: If sync fails, app continues offline
4. **Retry Logic**: Exponential backoff (1s, 2s, 4s, 8s, max 30s)
5. **Graceful Degradation**: Sync failures don't affect code generation

**Availability Calculation:**

- Single region: 99.5% uptime
- Multi-region (3 regions, any 2 up): $1 - (0.005)^2 = 99.9975%$
- Target: **99.9%** (achieved with 2+ regions)

### 7.3 Push Notification Service

**Requirement**: 99.9% uptime (push is optional, TOTP codes still work).

**Strategies:**

1. **FCM/APNs Redundancy**: Use Google/Apple's infrastructure (99.99% uptime)
2. **Fallback to TOTP**: If push fails, user can enter TOTP code
3. **Request Timeout**: 30-second timeout, fallback to TOTP
4. **Retry Logic**: Retry failed pushes (max 3 attempts)

**Result**: Push failures don't block authentication (TOTP fallback always available).

### 7.4 Data Loss Prevention

**Backup Strategies:**

1. **Local Backup**: Encrypted backup to device storage (automatic)
2. **Cloud Backup**: User-enabled encrypted cloud backup
3. **Export**: User can export accounts (encrypted file)

**Recovery:**

- **Device Lost**: Restore from cloud backup (if enabled)
- **Cloud Backup Lost**: Restore from local backup or re-add accounts
- **Both Lost**: User must re-add accounts manually (security feature)

---

## 8. Bottlenecks and Optimizations

### 8.1 Code Generation Performance

**Bottleneck**: HMAC-SHA1 computation (though already fast, ~1ms).

**Optimization**: Pre-compute codes for next 2 time steps (cache).

```
Current time: 10:00:00
Pre-compute:
  - Code for 10:00:00-10:00:30 (current)
  - Code for 10:00:30-10:01:00 (next)
  - Code for 10:01:00-10:01:30 (next+1)

When time reaches 10:00:30:
  - Use cached code (no computation)
  - Pre-compute code for 10:01:30
```

**Result**: Code display instant (0ms), computation happens in background.

### 8.2 Secret Decryption Performance

**Bottleneck**: HSM unlock + AES decryption (~50-100ms with biometric prompt).

**Optimization**: Cache decrypted secrets in memory (encrypted with session key).

```
1. User unlocks app (biometric) → HSM returns master key
2. Decrypt all secrets → Store in memory (encrypted with session key)
3. Session key stored in secure memory (not HSM, but encrypted)
4. Code generation uses cached secrets (no HSM unlock needed)
5. When app backgrounded → Clear session key, require biometric again
```

**Result**: First code generation: 100ms (HSM unlock), subsequent: <1ms (cached).

### 8.3 Sync Service Throughput

**Bottleneck**: 350 QPS sync operations (manageable, but can scale).

**Optimization**:

1. **Batch Sync**: Sync multiple accounts in one request
2. **Delta Sync**: Only sync changed accounts (not all accounts)
3. **CDN Caching**: Cache account icons/metadata (reduce server load)

**Result**: 350 QPS → 50 QPS (7x reduction via batching).

### 8.4 Push Notification Latency

**Bottleneck**: FCM/APNs latency (~500ms-2s).

**Optimization**: None (depends on Google/Apple infrastructure).

**Fallback**: TOTP codes always available (push is convenience, not requirement).

---

## 9. Common Anti-Patterns

### ❌ **1. Storing Secrets in Plaintext**

**Problem:**
```sql
-- BAD: Secrets stored unencrypted
CREATE TABLE accounts (
    secret TEXT  -- Plaintext secret!
);
```

**Why It's Bad:**
- Device compromise → attacker sees all secrets
- No security if device is lost/stolen

**Solution:**
```sql
-- GOOD: Encrypted with HSM
CREATE TABLE accounts (
    encrypted_secret TEXT,  -- AES-256-GCM encrypted
    encryption_nonce TEXT
);
```

### ❌ **2. Sending Secrets to Server**

**Problem:**
```
// BAD: Uploading plaintext secrets
POST /api/accounts
{
  "secret": "JBSWY3DPEHPK3PXP"  // Plaintext!
}
```

**Why It's Bad:**
- Server compromise → attacker sees all secrets
- Violates zero-knowledge principle

**Solution:**
```
// GOOD: Encrypt client-side before upload
POST /api/accounts
{
  "encrypted_secret": "AES-256-GCM(secret, cloud_key)",
  "nonce": "..."
}
// Server never sees plaintext
```

### ❌ **3. Generating Codes on Server**

**Problem:**
```
// BAD: Server generates codes
GET /api/code?account_id=123
→ Server generates TOTP code
→ Returns to client
```

**Why It's Bad:**
- Requires internet (doesn't work offline)
- Server compromise → attacker can generate codes
- High latency (network round-trip)

**Solution:**
```
// GOOD: Client generates codes locally
function generateCode(secret):
  return generate_totp(secret, current_timestamp())
// Works offline, no server dependency
```

### ❌ **4. No Time Drift Handling**

**Problem:**
```
// BAD: Only accepts exact time step
function validate(code, secret):
  expected = generate_totp(secret, current_timestamp())
  return code == expected  // Fails if clock drift > 30s
```

**Why It's Bad:**
- Device clock drift causes authentication failures
- Poor user experience

**Solution:**
```
// GOOD: Accept ±1 time step
function validate(code, secret):
  now = current_timestamp()
  steps = [now - 30, now, now + 30]
  for each step in steps:
    if generate_totp(secret, step) == code:
      return true
  return false
```

### ❌ **5. Weak Cloud Backup Encryption**

**Problem:**
```
// BAD: Simple encryption
encrypted = AES(account, password)  // No key derivation!
```

**Why It's Bad:**
- Weak passwords → easy to brute force
- No salt → rainbow table attacks

**Solution:**
```
// GOOD: PBKDF2 key derivation
salt = generate_random_bytes(16)
key = PBKDF2(password, salt, 100000, "SHA-256")
encrypted = AES_256_GCM(account, key, nonce)
```

---

## 10. Alternative Approaches

### 10.1 SMS-Based 2FA (Alternative to TOTP)

**How It Works:**
- Server generates 6-digit code
- Sends via SMS to user's phone
- User enters code on website

**Pros:**
- No app installation needed
- Works on any phone (SMS-capable)

**Cons:**
- **Requires Internet**: Doesn't work offline
- **SIM Swapping**: Attacker can hijack SMS
- **Latency**: 5-30 seconds (SMS delivery)
- **Cost**: SMS fees per code
- **Reliability**: SMS delivery failures

**When to Use**: Legacy systems, users without smartphones.

**Our Choice**: TOTP (offline, more secure, faster).

### 10.2 Hardware Tokens (YubiKey)

**How It Works:**
- Physical USB/NFC device
- Generates codes or provides cryptographic challenge-response
- No battery, no network

**Pros:**
- **Ultra-Secure**: Hardware tamper-resistant
- **Offline**: No device dependency
- **Fast**: <100ms response

**Cons:**
- **Cost**: $50-100 per device
- **Loss Risk**: Device lost = account locked
- **Compatibility**: Requires USB/NFC support

**When to Use**: High-security environments (enterprise, banking).

**Our Choice**: Software authenticator (free, convenient, good security).

### 10.3 Biometric-Only Authentication

**How It Works:**
- User authenticates with fingerprint/face
- No codes, no passwords

**Pros:**
- **Convenient**: No typing
- **Fast**: <1 second

**Cons:**
- **Privacy**: Biometric data stored (risky if compromised)
- **Reliability**: False positives/negatives
- **Device Dependency**: Requires biometric-capable device

**When to Use**: Low-security apps (unlocking phone).

**Our Choice**: TOTP + Biometric (unlock app with biometric, generate codes with TOTP).

---

## 11. Monitoring and Observability

### 11.1 Key Metrics

**Client-Side Metrics:**

- **Code Generation Latency**: <10ms (p99)
- **HSM Unlock Latency**: <200ms (p99, includes biometric)
- **App Crash Rate**: <0.1% of sessions
- **Offline Usage**: % of codes generated offline

**Server-Side Metrics:**

- **Sync Request Latency**: <100ms (p95)
- **Sync Success Rate**: >99.9%
- **Push Delivery Latency**: <2s (p95, FCM/APNs dependent)
- **Push Delivery Rate**: >99%
- **API Error Rate**: <0.1%

### 11.2 Alerts

**Critical Alerts:**

1. **Sync Service Down**: >5% error rate for 5 minutes
2. **Push Delivery Failure**: >10% failure rate for 5 minutes
3. **Database Latency**: >500ms (p95) for 10 minutes

**Warning Alerts:**

1. **High Sync Latency**: >200ms (p95) for 30 minutes
2. **Increased Error Rate**: >1% for 15 minutes

### 11.3 Logging

**Client Logs:**

- Account added/removed (anonymized)
- Code generation (no secrets logged)
- Sync operations (success/failure)
- App crashes (stack traces)

**Server Logs:**

- Sync requests (account IDs, timestamps)
- Push notifications (request IDs, delivery status)
- API errors (error codes, user IDs anonymized)

**Privacy**: Never log secrets, codes, or personally identifiable information.

---

## 12. Trade-offs Summary

| What You Gain | What You Sacrifice |
|---------------|-------------------|
| ✅ **Offline Operation** | ❌ No real-time account updates (eventual consistency) |
| ✅ **Zero-Knowledge Security** | ❌ Can't recover accounts if password forgotten |
| ✅ **Hardware Security (HSM)** | ❌ Requires biometric-capable device |
| ✅ **Multi-Device Sync** | ❌ Cloud backup requires user password (no reset) |
| ✅ **Push Notifications** | ❌ Requires internet (TOTP fallback available) |
| ✅ **Fast Code Generation** | ❌ Cached secrets in memory (cleared on background) |
| ✅ **Scalable (Client-Side)** | ❌ Limited server-side features (by design) |

**Key Trade-off**: **Security vs. Convenience**

- **More Secure**: Offline TOTP, zero-knowledge, HSM encryption
- **Less Convenient**: Manual account setup, password required for backup

**Decision**: Prioritize security (offline-first, zero-knowledge) over convenience (no password reset, manual setup).

---

## 13. Real-World Examples

### 13.1 Google Authenticator

**Architecture:**
- **Offline TOTP**: Codes generated locally (no internet)
- **No Cloud Backup**: Accounts stored only on device
- **QR Code Setup**: Scan QR to add accounts
- **Multi-Account**: Supports unlimited accounts

**Scale:**
- 500M+ users
- Billions of codes generated daily (client-side)
- Minimal server infrastructure (setup only)

**Key Design:**
- **Simplicity**: No cloud sync, no backup (security-focused)
- **Offline-First**: 100% offline code generation

### 13.2 Microsoft Authenticator

**Architecture:**
- **Offline TOTP**: Codes generated locally
- **Push Notifications**: Approve/deny login requests
- **Cloud Backup**: Encrypted backup to Microsoft account
- **Multi-Device**: Sync across devices

**Scale:**
- 200M+ users
- Millions of push approvals daily
- Azure infrastructure for sync/push

**Key Design:**
- **Convenience**: Push notifications, cloud backup
- **Security**: Encrypted backups, device registration

### 13.3 Authy

**Architecture:**
- **Offline TOTP**: Codes generated locally
- **Cloud Backup**: Encrypted backup (user password)
- **Multi-Device**: Sync across devices
- **SMS Backup**: Optional SMS codes (fallback)

**Scale:**
- 50M+ users
- Multi-device sync for millions of accounts

**Key Design:**
- **Recovery**: SMS backup codes (convenience)
- **Security**: Encrypted cloud backup

---

## 14. Deployment and Infrastructure

### 14.1 Client App Deployment

**iOS (App Store):**
- Native Swift app
- Uses iOS Keychain (Secure Enclave)
- Background App Refresh for sync
- APNs for push notifications

**Android (Google Play):**
- Native Kotlin app
- Uses Android Keystore (hardware-backed if available)
- WorkManager for background sync
- FCM for push notifications

### 14.2 Server Infrastructure

**Sync Service:**
- **Deployment**: Kubernetes (multi-region)
- **Scaling**: Auto-scale based on QPS (350 QPS baseline)
- **Database**: PostgreSQL (encrypted backups, device registrations)
- **Caching**: Redis (push tokens, sync state)

**Push Service:**
- **FCM Integration**: Google Firebase Cloud Messaging
- **APNs Integration**: Apple Push Notification Service
- **Queue**: RabbitMQ (push request queue, retry logic)

### 14.3 Security Infrastructure

**Encryption:**
- **At Rest**: AES-256-GCM (database encryption)
- **In Transit**: TLS 1.3 (all API calls)
- **Key Management**: AWS KMS / Azure Key Vault (server keys)

**Compliance:**
- **GDPR**: User data encryption, right to deletion
- **SOC 2**: Security controls, audit logs
- **Zero-Knowledge**: Server never sees plaintext secrets

---

## 15. Advanced Features

### 15.1 Watch App Support (Apple Watch / Wear OS)

**Challenge**: Limited storage, no HSM on watch.

**Solution:**
- **Encrypted Sync**: Watch receives encrypted secrets from phone
- **Limited Storage**: Cache 5-10 most-used accounts
- **Biometric Unlock**: Watch unlock (if supported)
- **Code Display**: Show codes on watch face

**Flow:**
```
1. Phone encrypts secrets with watch-specific key
2. Watch receives encrypted secrets via Bluetooth
3. Watch decrypts (requires watch unlock)
4. Watch generates codes locally
5. Codes displayed on watch
```

### 15.2 Account Organization

**Features:**
- **Groups**: Organize accounts by category (Work, Personal, Banking)
- **Search**: Find accounts by name/issuer
- **Favorites**: Pin frequently used accounts
- **Custom Icons**: User-uploaded icons for accounts

**Implementation:**
- Local database (SQLite) with groups table
- Sync groups/metadata to cloud (encrypted)
- Search index (FTS5 for SQLite)

### 15.3 Export/Import

**Export:**
- Generate encrypted backup file (user password)
- User downloads file (local storage or cloud)
- File contains all accounts (encrypted)

**Import:**
- User uploads backup file
- Enter password → decrypt accounts
- Add accounts to app (requires user approval)

**Security**: Export file encrypted with user password (PBKDF2, 100k iterations).

---

## 16. Interview Discussion Points

### 16.1 "How does TOTP work offline?"

**Answer:**
- TOTP is deterministic: same secret + same time = same code
- Device computes code locally using current time (no server needed)
- Server validates code using same algorithm (synchronized time)
- Time window: ±1 time step (90 seconds) handles clock drift

### 16.2 "How do you sync accounts across devices securely?"

**Answer:**
- Encrypt accounts with user's cloud password (PBKDF2 key derivation)
- Upload encrypted accounts to server (zero-knowledge)
- Other devices download and decrypt (same password)
- Each device re-encrypts with its own HSM master key
- Server never sees plaintext secrets

### 16.3 "What happens if the device is lost?"

**Answer:**
- **With Cloud Backup**: User installs app on new device → enters password → restores accounts
- **Without Cloud Backup**: User must re-add accounts manually (scan QR codes again)
- **Security**: Lost device can't decrypt secrets without biometric/device HSM (secrets encrypted)

### 16.4 "How do push notifications work?"

**Answer:**
- Website sends approval request to authenticator service
- Service sends push notification to user's registered devices (FCM/APNs)
- User taps "Approve" or "Deny" on phone
- App sends response to service → service notifies website
- Timeout: 30 seconds, fallback to TOTP code if push fails

### 16.5 "What if the server is down?"

**Answer:**
- **Code Generation**: Works offline (no server dependency)
- **Sync**: Fails gracefully (codes still work, sync retries later)
- **Push Notifications**: Falls back to TOTP codes (user enters code manually)
- **Result**: Core functionality (code generation) always available

---

## 17. References

### Related System Design Components

- **[2.4.1 Security Fundamentals](../02-components/2.4-security-observability/2.4.1-security-fundamentals.md)** - Encryption, hashing, authentication
- **[2.4.4 OAuth & JWT Deep Dive](../02-components/2.4-security-observability/2.4.4-oauth-jwt-deep-dive.md)** - Authentication protocols
- **[2.2.1 Caching Deep Dive](../02-components/2.2-caching/2.2.1-caching-deep-dive.md)** - Client-side caching strategies
- **[1.1.7 Idempotency](../01-principles/1.1.7-idempotency.md)** - Idempotent operations

### Related Design Challenges

- **[3.5.1 Payment Gateway](./3.5.1-payment-gateway/README.md)** - Security, encryption, compliance
- **[3.2.2 Notification Service](./3.2.2-notification-service/README.md)** - Push notifications, multi-device delivery

### External Resources

- **RFC 6238**: TOTP Algorithm Specification
- **RFC 4226**: HOTP Algorithm (counter-based, predecessor to TOTP)
- **Google Authenticator**: Open-source implementation
- **OWASP MFA**: Multi-factor authentication best practices

### Books

- **"Applied Cryptography"** by Bruce Schneier - Cryptography fundamentals
- **"Security Engineering"** by Ross Anderson - Security architecture

---

## 18. Summary

**Key Takeaways:**

1. **Offline-First Design**: TOTP codes generated locally (no internet required)
2. **Zero-Knowledge Architecture**: Server never sees plaintext secrets (encrypted client-side)
3. **Hardware Security**: Use device HSM (Keychain/Keystore) for key encryption
4. **Multi-Device Sync**: Encrypted cloud backup with user-controlled password
5. **Time Synchronization**: Accept ±1 time step (90-second window) for clock drift
6. **Push Notifications**: Optional convenience feature (TOTP fallback always available)
7. **Scalability**: Most operations client-side (minimal server infrastructure needed)

**Design Philosophy**: **Security over convenience** - offline operation, zero-knowledge, hardware encryption prioritize security, even if it means manual setup and no password reset.

