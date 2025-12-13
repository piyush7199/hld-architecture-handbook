# Authenticator App - Sequence Diagrams

## Table of Contents

1. [Offline Code Generation Flow](#offline-code-generation-flow)
2. [Background Sync Flow](#background-sync-flow)
3. [QR Code Scanning Flow](#qr-code-scanning-flow)
4. [Push Notification Approval Flow](#push-notification-approval-flow)
5. [Backup and Restore Flow](#backup-and-restore-flow)

---

## Offline Code Generation Flow

**Flow:**

Shows the complete sequence of generating TOTP codes offline, from user opening the app to displaying codes, with no network calls required.

**Steps:**

1. **User Opens App** (0ms): User launches authenticator app
2. **UI Requests Codes** (10ms): UI layer requests codes for all accounts
3. **HSM Unlock Request** (20ms): TOTP engine requests master key from device HSM
4. **Biometric Prompt** (50ms): HSM prompts user for biometric authentication
5. **Biometric Auth** (500ms): User authenticates with Face ID / fingerprint
6. **Master Key Returned** (550ms): HSM returns temporary master key to TOTP engine
7. **Read Encrypted Secrets** (560ms): TOTP engine reads encrypted secrets from local database
8. **Secrets Decrypted** (600ms): HSM decrypts secrets using master key
9. **TOTP Generation** (610ms): TOTP engine generates codes using HMAC-SHA1 algorithm
10. **Codes Displayed** (620ms): UI displays 6-digit codes to user
11. **Auto-Refresh** (30s): Codes automatically refresh every 30 seconds

**Performance:**

- Total latency: <1 second (mostly biometric prompt)
- Code generation: <10ms (after HSM unlock)
- Works completely offline (no network calls)

**Key Points:**

- All operations are local (offline-first design)
- Secrets decrypted only when needed (HSM protected)
- Codes auto-refresh every 30 seconds

```mermaid
sequenceDiagram
    participant User
    participant UI as UI Layer
    participant TOTP as TOTP Engine
    participant DB as Local Database
    participant HSM as Device HSM
    
    User->>UI: Open App
    Note over UI: t=0ms
    UI->>TOTP: Request Codes
    Note over TOTP: t=10ms
    TOTP->>HSM: Request Master Key
    Note over HSM: t=20ms
    HSM->>User: Biometric Prompt<br/>Face ID / Fingerprint
    Note over User: t=50ms
    User->>HSM: Biometric Auth
    Note over HSM: t=500ms
    HSM->>TOTP: Master Key (temporary)
    Note over TOTP: t=550ms
    TOTP->>DB: Read Encrypted Secrets
    Note over DB: t=560ms
    DB->>TOTP: Encrypted Secrets
    TOTP->>HSM: Decrypt Secrets
    Note over HSM: t=600ms
    HSM->>TOTP: Plaintext Secrets
    TOTP->>TOTP: Generate TOTP Codes<br/>HMAC-SHA1(secret, time)
    Note over TOTP: t=610ms
    TOTP->>UI: 6-Digit Codes
    UI->>User: Display Codes
    Note over UI: t=620ms
    Note over TOTP: Auto-Refresh Every 30s
```

---

## Background Sync Flow

**Flow:**

Shows the background synchronization process when the app is online, updating account metadata (names, icons, issuer info) without affecting offline code generation.

**Steps:**

1. **Connectivity Detection** (0ms): App detects internet connection available
2. **Sync Trigger** (100ms): Background sync service triggers (every 6 hours, or on app open)
3. **Sync Request** (200ms): App sends account IDs and last sync timestamps to server
4. **Server Query** (250ms): Server queries metadata database for latest account information
5. **Metadata Retrieved** (300ms): Database returns latest metadata (names, icons, issuer)
6. **Change Detection** (350ms): Server compares client timestamps with server data
7. **Response Sent** (400ms): Server returns updates, deleted accounts, new accounts
8. **Local Update** (500ms): App updates local database with new metadata
9. **Sync Complete** (600ms): App updates sync timestamps

**Performance:**

- Total latency: <1 second
- Server processing: <200ms
- Works in background (non-blocking)

**Privacy:**

- Server only sees account IDs and timestamps
- Secrets never sent to server (zero-knowledge architecture)
- Only metadata is synchronized

**Key Points:**

- Non-blocking: Sync happens in background (codes work offline)
- Efficient: Only syncs changed accounts (delta sync)
- Privacy-preserving: Server never sees secrets

```mermaid
sequenceDiagram
    participant App as Authenticator App
    participant Sync as Background Sync Service
    participant API as Sync API
    participant DB as Server Database
    
    App->>Sync: Detect Internet Connection
    Note over Sync: t=0ms
    Sync->>Sync: Trigger Sync<br/>(Every 6 hours or on app open)
    Note over Sync: t=100ms
    Sync->>API: POST /sync<br/>{account_ids: [1,2,3], last_sync: {...}}
    Note over API: t=200ms
    API->>DB: Query Metadata<br/>SELECT * FROM accounts WHERE id IN (...)
    Note over DB: t=250ms
    DB->>API: Latest Metadata<br/>{account_1: {name, icon, issuer}, ...}
    Note over API: t=300ms
    API->>API: Compare Changes<br/>Detect updates, deleted, new
    Note over API: t=350ms
    API->>Sync: Response:<br/>{updates: [...], deleted: [...], new: [...]}
    Note over Sync: t=400ms
    Sync->>App: Update Local Database
    Note over App: t=500ms
    App->>Sync: Sync Complete<br/>Update sync timestamps
    Note over Sync: t=600ms
```

---

## QR Code Scanning Flow

**Flow:**

Shows the complete flow of scanning a QR code to add an account to the authenticator app, including camera access, QR parsing, validation, encryption, and storage.

**Steps:**

1. **User Initiates** (0ms): User taps "Add Account" → "Scan QR Code"
2. **Camera Permission** (100ms): App requests camera permission (if not granted)
3. **Permission Granted** (500ms): User grants camera permission
4. **QR Detection** (1000ms): Camera detects QR code pattern
5. **QR Decode** (1100ms): Extract text string from QR code (otpauth:// URL)
6. **URL Validation** (1150ms): Validate URL starts with `otpauth://totp/`
7. **Parse Parameters** (1200ms): Extract secret, issuer, account name, algorithm, digits, period
8. **Secret Validation** (1250ms): Validate secret is Base32 encoded, correct length
9. **HSM Encryption Request** (1300ms): Request HSM master key for encryption
10. **Biometric Prompt** (1350ms): HSM prompts user for biometric authentication
11. **Biometric Auth** (1800ms): User authenticates with Face ID / fingerprint
12. **Secret Encrypted** (1850ms): HSM encrypts secret with master key (AES-256-GCM)
13. **Account Saved** (1900ms): Save encrypted account to local database
14. **Code Verification** (1950ms): Generate first TOTP code to verify account works
15. **Account Displayed** (2000ms): Display new account in account list

**Performance:**

- Total latency: ~2 seconds (mostly camera detection and biometric)
- QR detection: <500ms (depends on camera quality)
- Encryption: <50ms (HSM operation)

**Error Handling:**

- Invalid QR code: Show error "Invalid QR code format"
- Duplicate account: Detect if account already exists
- Camera permission denied: Prompt user to grant permission
- HSM unavailable: Fallback to software encryption (less secure)

**Key Points:**

- Fast setup: One-tap account addition
- Secure: Secret encrypted immediately with HSM
- Validation: Verify account works before saving
- No network: All operations local

```mermaid
sequenceDiagram
    participant User
    participant UI as App UI
    participant Camera
    participant Parser as QR Parser
    participant Validator as Secret Validator
    participant HSM as Device HSM
    participant DB as Local Database
    participant TOTP as TOTP Engine
    
    User->>UI: Tap "Add Account" → "Scan QR"
    Note over UI: t=0ms
    UI->>Camera: Request Camera Access
    Note over Camera: t=100ms
    Camera->>User: Permission Prompt
    User->>Camera: Grant Permission
    Note over Camera: t=500ms
    Camera->>UI: Camera Active
    
    User->>Camera: Point at QR Code
    Camera->>Parser: QR Code Detected<br/>Text: "otpauth://totp/..."
    Note over Parser: t=1000ms
    Parser->>Parser: Parse URL<br/>Extract: secret, issuer, name
    Note over Parser: t=1100ms
    Parser->>Validator: Validate Secret<br/>Base32, length check
    Note over Validator: t=1250ms
    Validator->>Parser: Valid: True
    
    Parser->>HSM: Encrypt Secret<br/>Request master key
    Note over HSM: t=1300ms
    HSM->>User: Biometric Prompt<br/>Face ID / Fingerprint
    Note over User: t=1350ms
    User->>HSM: Biometric Auth
    Note over HSM: t=1800ms
    HSM->>Parser: Encrypted Secret<br/>AES-256-GCM
    Note over Parser: t=1850ms
    
    Parser->>DB: Save Account<br/>{encrypted_secret, issuer, name}
    Note over DB: t=1900ms
    DB->>Parser: Account Saved<br/>account_id: 123
    
    Parser->>TOTP: Generate First Code<br/>Verify account works
    TOTP->>TOTP: HMAC-SHA1(secret, time)
    Note over TOTP: t=1950ms
    TOTP->>Parser: Code: 847362
    
    Parser->>UI: Account Added<br/>Display in list
    UI->>User: Show New Account<br/>Code: 847362
    Note over UI: t=2000ms
    
    Note over UI,DB: Optional: Upload to cloud backup
```

---

## Push Notification Approval Flow

**Flow:**

Shows how push notification approvals work (Microsoft Authenticator style), allowing users to approve login requests with a tap instead of entering TOTP codes.

**Steps:**

1. **User Login Attempt** (0ms): User attempts login on website
2. **Approval Request** (100ms): Website sends approval request to authenticator service
3. **Device Lookup** (150ms): Service looks up user's registered devices from database
4. **Push Notification Sent** (200ms): Service sends push notification via FCM/APNs
5. **Notification Received** (500ms): User's device receives push notification
6. **Notification Displayed** (600ms): User sees "Approve login?" notification
7. **User Action** (5000ms): User taps "Approve" or "Deny" (user time)
8. **Response Sent** (5100ms): App sends approval/denial to service
9. **Website Notification** (5200ms): Service notifies website via WebSocket
10. **Login Complete** (5300ms): Website completes or rejects authentication

**Performance:**

- End-to-end latency: <6 seconds (mostly user response time)
- Push delivery: <500ms (FCM/APNs)
- WebSocket notification: <100ms

**Security:**

- 30-second timeout (prevents replay attacks)
- Device verification (only registered devices)
- Rate limiting (max 5 requests per 15 minutes)
- Biometric confirmation (optional, for sensitive accounts)

**Key Points:**

- Convenient: One-tap approval (no typing codes)
- Fast: <2 seconds server processing
- Fallback: TOTP codes always available if push fails
- Secure: Timeout and device verification

```mermaid
sequenceDiagram
    participant User
    participant Website
    participant Service as Authenticator Service
    participant DB as Device Database
    participant FCM as FCM/APNs
    participant App as Mobile App
    
    User->>Website: Attempt Login<br/>username + password
    Note over Website: t=0ms
    Website->>Service: Approval Request<br/>{user_id, session_id, ip}
    Note over Service: t=100ms
    Service->>DB: Lookup Registered Devices<br/>SELECT devices WHERE user_id = ...
    Note over DB: t=150ms
    DB->>Service: Device List<br/>[{device_id, push_token}, ...]
    Service->>FCM: Send Push Notification<br/>{token, title: "Approve login?", ...}
    Note over FCM: t=200ms
    FCM->>App: Push Notification Received
    Note over App: t=500ms
    App->>User: Show Notification<br/>"Approve login? [Approve] [Deny]"
    Note over User: t=600ms
    User->>App: Tap "Approve"
    Note over User: t=5000ms (user time)
    App->>Service: POST /approve<br/>{session_id, action: "approve"}
    Note over Service: t=5100ms
    Service->>Website: WebSocket Message<br/>{session_id, status: "approved"}
    Note over Website: t=5200ms
    Website->>User: Login Successful<br/>Redirect to dashboard
    Note over Website: t=5300ms
    
    alt User Denies
        User->>App: Tap "Deny"
        App->>Service: POST /approve<br/>{session_id, action: "deny"}
        Service->>Website: WebSocket Message<br/>{session_id, status: "denied"}
        Website->>User: Login Rejected
    end
    
    alt Timeout (30 seconds)
        Note over Service: Timeout after 30s
        Service->>Website: WebSocket Message<br/>{session_id, status: "timeout"}
        Website->>User: Request Expired
    end
```

---

## Backup and Restore Flow

**Flow:**

Shows how users backup their accounts to the cloud and restore them on a new device, using zero-knowledge encryption where the server never sees plaintext secrets.

**Backup Steps:**

1. **User Enables Backup** (0ms): User enables cloud backup in settings
2. **Password Entry** (1000ms): User enters backup password
3. **Key Derivation** (1500ms): PBKDF2(password, salt, 100k iterations) → cloud encryption key
4. **Account Encryption** (2000ms): Encrypt each account with cloud key (separate from HSM encryption)
5. **Upload Request** (2500ms): Prepare encrypted accounts blob for upload
6. **Server Storage** (3000ms): Server stores encrypted data (never sees plaintext)
7. **Backup Complete** (3500ms): User receives confirmation

**Restore Steps:**

1. **New Device Setup** (0ms): User installs app on new device
2. **Restore Option** (500ms): User selects "Restore from Backup"
3. **Password Entry** (2000ms): User enters same backup password
4. **Key Derivation** (2500ms): Derive same cloud key using PBKDF2
5. **Download Request** (3000ms): Request encrypted accounts from server
6. **Server Response** (3500ms): Server returns encrypted accounts blob
7. **Decryption** (4000ms): Decrypt accounts using cloud key
8. **Re-Encryption** (4500ms): Re-encrypt with new device's HSM master key
9. **Storage** (5000ms): Save accounts to new device's local database
10. **Verification** (5500ms): Generate codes to verify restore successful

**Performance:**

- Key derivation: ~500ms (PBKDF2, 100k iterations)
- Account encryption: <50ms per account
- Upload/download: Depends on network (typically <2 seconds)
- Total restore: <6 seconds for 10 accounts

**Security:**

- Zero-knowledge: Server never sees plaintext secrets
- User-controlled: User controls encryption key (password)
- No password reset: If password lost, backup cannot be decrypted (by design)
- Device-specific: Each device has its own HSM master key

**Key Points:**

- Account recovery: Restore accounts if device lost
- Multi-device: Same accounts on all devices
- Privacy: Server cannot decrypt accounts
- User control: User owns encryption key

```mermaid
sequenceDiagram
    participant User
    participant App1 as Device 1 App
    participant HSM1 as Device 1 HSM
    participant Server as Backup Server
    participant App2 as Device 2 App
    participant HSM2 as Device 2 HSM
    
    Note over User,HSM1: Backup Process
    User->>App1: Enable Cloud Backup
    Note over App1: t=0ms
    App1->>User: Enter Backup Password
    User->>App1: Password: "mypassword123"
    Note over App1: t=1000ms
    App1->>App1: PBKDF2 Key Derivation<br/>100k iterations
    Note over App1: t=1500ms
    App1->>HSM1: Request Accounts<br/>Decrypt with HSM key
    HSM1->>App1: Plaintext Accounts
    App1->>App1: Encrypt with Cloud Key<br/>AES-256-GCM
    Note over App1: t=2000ms
    App1->>Server: Upload Encrypted Accounts<br/>POST /backup {encrypted_blob}
    Note over Server: t=2500ms
    Server->>Server: Store Encrypted Data<br/>(Never sees plaintext)
    Note over Server: t=3000ms
    Server->>App1: Backup Complete<br/>200 OK
    Note over App1: t=3500ms
    
    Note over User,HSM2: Restore Process (New Device)
    User->>App2: Install App → "Restore from Backup"
    Note over App2: t=0ms
    App2->>User: Enter Backup Password
    User->>App2: Password: "mypassword123"
    Note over App2: t=2000ms
    App2->>App2: PBKDF2 Key Derivation<br/>Same Key
    Note over App2: t=2500ms
    App2->>Server: Download Encrypted Accounts<br/>GET /backup
    Note over Server: t=3000ms
    Server->>App2: Encrypted Accounts Blob
    Note over App2: t=3500ms
    App2->>App2: Decrypt with Cloud Key
    Note over App2: t=4000ms
    App2->>HSM2: Re-Encrypt with HSM Key 2<br/>Request master key
    HSM2->>User: Biometric Prompt
    User->>HSM2: Biometric Auth
    HSM2->>App2: Encrypted Accounts<br/>(Device 2 HSM key)
    Note over App2: t=4500ms
    App2->>App2: Save to Local Database
    Note over App2: t=5000ms
    App2->>App2: Generate Codes<br/>Verify restore successful
    Note over App2: t=5500ms
    App2->>User: Restore Complete<br/>Accounts Available
```

