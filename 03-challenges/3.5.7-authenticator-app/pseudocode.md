# Authenticator App - Pseudocode Implementations

This document contains detailed algorithm implementations for the Authenticator App system. The main challenge document references these functions.

---

## Table of Contents

1. [TOTP Code Generation](#totp-code-generation)
2. [Secret Encryption and Decryption](#secret-encryption-and-decryption)
3. [QR Code Parsing and Account Setup](#qr-code-parsing-and-account-setup)
4. [Background Synchronization](#background-synchronization)
5. [Cloud Backup and Restore](#cloud-backup-and-restore)
6. [Push Notification Approvals](#push-notification-approvals)
7. [Time Synchronization and Drift Handling](#time-synchronization-and-drift-handling)

---

## TOTP Code Generation

### generate_totp()

Generates a time-based one-time password (TOTP) using RFC 6238 algorithm.

**Parameters:**

- `secret` (string): Base32-encoded secret key
- `timestamp` (integer, optional): Unix timestamp (defaults to current time)
- `algorithm` (string, optional): Hash algorithm - "SHA1", "SHA256", or "SHA512" (default: "SHA1")
- `digits` (integer, optional): Number of digits in code (default: 6)
- `period` (integer, optional): Time step in seconds (default: 30)

**Returns:**

- `string`: 6-digit TOTP code (e.g., "847362")

**Algorithm:**

```
function generate_totp(secret, timestamp = None, algorithm = "SHA1", digits = 6, period = 30):
    // Get current time if not provided
    if timestamp == None:
        timestamp = current_unix_timestamp()
    
    // Decode Base32 secret
    secret_bytes = base32_decode(secret)
    
    // Calculate time step (counter)
    time_step = timestamp / period  // integer division
    
    // Convert time step to 8-byte big-endian
    time_step_bytes = int_to_bytes_big_endian(time_step, 8)
    
    // Compute HMAC
    if algorithm == "SHA1":
        hmac_hash = hmac_sha1(secret_bytes, time_step_bytes)
    else if algorithm == "SHA256":
        hmac_hash = hmac_sha256(secret_bytes, time_step_bytes)
    else if algorithm == "SHA512":
        hmac_hash = hmac_sha512(secret_bytes, time_step_bytes)
    
    // Dynamic truncation (RFC 4226)
    offset = hmac_hash[19] & 0x0F  // Last byte, lower 4 bits
    binary_code = (hmac_hash[offset] & 0x7F) << 24
    binary_code = binary_code | ((hmac_hash[offset + 1] & 0xFF) << 16)
    binary_code = binary_code | ((hmac_hash[offset + 2] & 0xFF) << 8)
    binary_code = binary_code | (hmac_hash[offset + 3] & 0xFF)
    
    // Generate 6-digit code
    code = binary_code % (10 ^ digits)
    
    // Format as string with leading zeros
    return format(code, "0" + digits)
```

**Time Complexity:** O(1)

**Example Usage:**

```
secret = "JBSWY3DPEHPK3PXP"
code = generate_totp(secret)
// Output: "847362" (valid for 30 seconds)
```

---

## Secret Encryption and Decryption

### encrypt_secret()

Encrypts a secret key using device HSM master key.

**Parameters:**

- `plaintext_secret` (string): Plaintext secret key to encrypt
- `master_key` (bytes): Device HSM master key (from Keychain/Keystore)

**Returns:**

- `string`: Base64-encoded encrypted secret (AES-256-GCM)

**Algorithm:**

```
function encrypt_secret(plaintext_secret, master_key):
    // Generate random IV (12 bytes for GCM)
    iv = generate_random_bytes(12)
    
    // Convert secret to bytes
    secret_bytes = plaintext_secret.encode('utf-8')
    
    // Encrypt using AES-256-GCM
    cipher = AES_GCM(master_key, iv)
    encrypted_data, auth_tag = cipher.encrypt_and_authenticate(secret_bytes)
    
    // Combine IV + encrypted_data + auth_tag
    encrypted_blob = iv + encrypted_data + auth_tag
    
    // Encode as Base64 for storage
    return base64_encode(encrypted_blob)
```

**Time Complexity:** O(1)

### decrypt_secret()

Decrypts an encrypted secret using device HSM master key.

**Parameters:**

- `encrypted_secret` (string): Base64-encoded encrypted secret
- `master_key` (bytes): Device HSM master key

**Returns:**

- `string`: Plaintext secret key

**Algorithm:**

```
function decrypt_secret(encrypted_secret, master_key):
    // Decode from Base64
    encrypted_blob = base64_decode(encrypted_secret)
    
    // Extract components
    iv = encrypted_blob[0:12]
    auth_tag = encrypted_blob[-16:]
    encrypted_data = encrypted_blob[12:-16]
    
    // Decrypt using AES-256-GCM
    cipher = AES_GCM(master_key, iv)
    plaintext_bytes = cipher.decrypt_and_verify(encrypted_data, auth_tag)
    
    // Convert to string
    return plaintext_bytes.decode('utf-8')
```

**Time Complexity:** O(1)

**Example Usage:**

```
master_key = hsm_get_master_key()  // From Keychain/Keystore
encrypted = encrypt_secret("JBSWY3DPEHPK3PXP", master_key)
decrypted = decrypt_secret(encrypted, master_key)
// decrypted == "JBSWY3DPEHPK3PXP"
```

---

## QR Code Parsing and Account Setup

### parse_otpauth_url()

Parses an otpauth:// URL from QR code and extracts account information.

**Parameters:**

- `url` (string): otpauth:// URL (e.g., "otpauth://totp/Issuer:Account?secret=...")

**Returns:**

- `object`: Account information with fields: secret, issuer, account_name, algorithm, digits, period

**Algorithm:**

```
function parse_otpauth_url(url):
    // Validate URL format
    if not url.starts_with("otpauth://totp/"):
        throw InvalidURLException("URL must start with otpauth://totp/")
    
    // Parse URL components
    parts = url.split("?")
    path = parts[0]  // "otpauth://totp/Issuer:Account"
    query = parts[1] if len(parts) > 1 else ""  // "secret=...&issuer=..."
    
    // Extract issuer and account name from path
    path_parts = path.replace("otpauth://totp/", "").split(":")
    issuer = path_parts[0] if len(path_parts) > 0 else ""
    account_name = path_parts[1] if len(path_parts) > 1 else ""
    
    // Parse query parameters
    params = parse_query_string(query)
    
    // Extract secret (required)
    secret = params.get("secret")
    if not secret:
        throw InvalidURLException("Secret is required")
    
    // Validate secret is Base32
    if not is_valid_base32(secret):
        throw InvalidURLException("Secret must be Base32 encoded")
    
    // Extract optional parameters
    algorithm = params.get("algorithm", "SHA1")  // SHA1, SHA256, SHA512
    digits = int(params.get("digits", "6"))
    period = int(params.get("period", "30"))
    
    // Override issuer from query if present
    if params.get("issuer"):
        issuer = params.get("issuer")
    
    return {
        "secret": secret,
        "issuer": issuer,
        "account_name": account_name,
        "algorithm": algorithm,
        "digits": digits,
        "period": period
    }
```

**Time Complexity:** O(n) where n is URL length

### add_account()

Adds a new account to the authenticator app.

**Parameters:**

- `account_info` (object): Account information from parse_otpauth_url()
- `master_key` (bytes): Device HSM master key

**Returns:**

- `string`: Account ID (UUID)

**Algorithm:**

```
function add_account(account_info, master_key):
    // Validate secret
    if not is_valid_base32(account_info.secret):
        throw InvalidSecretException("Secret must be Base32 encoded")
    
    if len(account_info.secret) < 16:
        throw InvalidSecretException("Secret too short (minimum 16 characters)")
    
    // Check for duplicate account
    existing = database.find_account(
        issuer = account_info.issuer,
        account_name = account_info.account_name
    )
    if existing:
        throw DuplicateAccountException("Account already exists")
    
    // Encrypt secret
    encrypted_secret = encrypt_secret(account_info.secret, master_key)
    
    // Generate account ID
    account_id = generate_uuid()
    
    // Create account record
    account_record = {
        "account_id": account_id,
        "issuer": account_info.issuer,
        "account_name": account_info.account_name,
        "encrypted_secret": encrypted_secret,
        "algorithm": account_info.algorithm,
        "digits": account_info.digits,
        "period": account_info.period,
        "added_at": current_timestamp(),
        "last_used": current_timestamp()
    }
    
    // Save to database
    database.insert_account(account_record)
    
    // Verify account works by generating first code
    code = generate_totp(
        account_info.secret,
        algorithm = account_info.algorithm,
        digits = account_info.digits,
        period = account_info.period
    )
    
    // Log verification
    logger.info("Account added and verified", account_id = account_id, code = code)
    
    return account_id
```

**Time Complexity:** O(1)

**Example Usage:**

```
url = "otpauth://totp/Google:user@example.com?secret=JBSWY3DPEHPK3PXP&issuer=Google"
account_info = parse_otpauth_url(url)
master_key = hsm_get_master_key()
account_id = add_account(account_info, master_key)
```

---

## Background Synchronization

### sync_accounts()

Synchronizes account metadata with server.

**Parameters:**

- `account_ids` (array): List of account IDs to sync
- `last_sync_timestamps` (object): Map of account_id -> last_sync_timestamp

**Returns:**

- `object`: Sync result with updates, deleted, and new accounts

**Algorithm:**

```
function sync_accounts(account_ids, last_sync_timestamps):
    // Check internet connectivity
    if not is_online():
        throw NetworkException("No internet connection")
    
    // Prepare sync request
    sync_request = {
        "account_ids": account_ids,
        "last_sync": last_sync_timestamps,
        "device_id": get_device_id(),
        "app_version": get_app_version()
    }
    
    // Send sync request to server
    try:
        response = http_post("/api/sync", sync_request)
    except NetworkException:
        throw NetworkException("Sync request failed")
    
    // Process response
    updates = []
    deleted = []
    new_accounts = []
    
    // Update existing accounts
    for update in response.updates:
        account = database.find_account(update.account_id)
        if account:
            // Update metadata only (not secret)
            account.issuer = update.issuer
            account.account_name = update.account_name
            account.icon_url = update.icon_url
            account.last_sync = current_timestamp()
            database.update_account(account)
            updates.append(update.account_id)
    
    // Handle deleted accounts
    for account_id in response.deleted:
        account = database.find_account(account_id)
        if account:
            database.delete_account(account_id)
            deleted.append(account_id)
    
    // Handle new accounts (metadata only, secret must be added locally)
    for new_account in response.new_accounts:
        // Only add if account doesn't exist locally
        existing = database.find_account(
            issuer = new_account.issuer,
            account_name = new_account.account_name
        )
        if not existing:
            // Create account record (without secret - user must scan QR)
            account_record = {
                "account_id": new_account.account_id,
                "issuer": new_account.issuer,
                "account_name": new_account.account_name,
                "icon_url": new_account.icon_url,
                "encrypted_secret": null,  // Secret must be added locally
                "needs_setup": true,
                "added_at": current_timestamp()
            }
            database.insert_account(account_record)
            new_accounts.append(new_account.account_id)
    
    // Update sync timestamps
    for account_id in account_ids:
        last_sync_timestamps[account_id] = current_timestamp()
    
    return {
        "updates": updates,
        "deleted": deleted,
        "new_accounts": new_accounts,
        "sync_timestamp": current_timestamp()
    }
```

**Time Complexity:** O(n) where n is number of accounts

**Example Usage:**

```
account_ids = ["uuid-1", "uuid-2", "uuid-3"]
last_sync = {
    "uuid-1": "2024-01-01T00:00:00Z",
    "uuid-2": "2024-01-01T00:00:00Z",
    "uuid-3": "2024-01-01T00:00:00Z"
}
result = sync_accounts(account_ids, last_sync)
```

---

## Cloud Backup and Restore

### backup_to_cloud()

Backs up accounts to cloud with zero-knowledge encryption.

**Parameters:**

- `backup_password` (string): User's backup password
- `account_ids` (array, optional): Account IDs to backup (default: all accounts)

**Returns:**

- `string`: Backup ID

**Algorithm:**

```
function backup_to_cloud(backup_password, account_ids = None):
    // Get all accounts if not specified
    if account_ids == None:
        accounts = database.get_all_accounts()
    else:
        accounts = [database.find_account(id) for id in account_ids]
    
    // Generate random salt
    salt = generate_random_bytes(32)
    
    // Derive cloud encryption key from password
    cloud_key = pbkdf2(
        password = backup_password,
        salt = salt,
        iterations = 100000,
        key_length = 32,
        hash_algorithm = "SHA256"
    )
    
    // Prepare accounts for backup (decrypt with HSM, re-encrypt with cloud key)
    master_key = hsm_get_master_key()
    backup_accounts = []
    
    for account in accounts:
        // Decrypt secret with HSM key
        plaintext_secret = decrypt_secret(account.encrypted_secret, master_key)
        
        // Re-encrypt with cloud key
        cloud_encrypted = encrypt_with_key(plaintext_secret, cloud_key)
        
        // Create backup record (no plaintext secrets)
        backup_account = {
            "account_id": account.account_id,
            "issuer": account.issuer,
            "account_name": account.account_name,
            "encrypted_secret": cloud_encrypted,
            "algorithm": account.algorithm,
            "digits": account.digits,
            "period": account.period,
            "icon_url": account.icon_url
        }
        backup_accounts.append(backup_account)
    
    // Create backup blob
    backup_blob = {
        "version": "1.0",
        "salt": base64_encode(salt),
        "accounts": backup_accounts,
        "backup_timestamp": current_timestamp(),
        "device_id": get_device_id()
    }
    
    // Encrypt backup blob
    encrypted_backup = encrypt_with_key(
        json_encode(backup_blob),
        cloud_key
    )
    
    // Upload to server
    backup_id = generate_uuid()
    try:
        response = http_post("/api/backup", {
            "backup_id": backup_id,
            "encrypted_data": base64_encode(encrypted_backup)
        })
    except NetworkException:
        throw NetworkException("Backup upload failed")
    
    // Store backup ID locally
    database.save_backup_id(backup_id, current_timestamp())
    
    return backup_id
```

**Time Complexity:** O(n) where n is number of accounts

### restore_from_cloud()

Restores accounts from cloud backup.

**Parameters:**

- `backup_password` (string): User's backup password
- `backup_id` (string, optional): Specific backup ID (default: latest)

**Returns:**

- `integer`: Number of accounts restored

**Algorithm:**

```
function restore_from_cloud(backup_password, backup_id = None):
    // Check internet connectivity
    if not is_online():
        throw NetworkException("No internet connection")
    
    // Download backup from server
    try:
        if backup_id:
            response = http_get("/api/backup/" + backup_id)
        else:
            response = http_get("/api/backup/latest")
    except NetworkException:
        throw NetworkException("Backup download failed")
    
    encrypted_backup = base64_decode(response.encrypted_data)
    
    // Extract salt from backup (first 32 bytes after version)
    // Note: In practice, salt is stored separately in backup metadata
    salt = base64_decode(response.salt)
    
    // Derive cloud encryption key from password
    cloud_key = pbkdf2(
        password = backup_password,
        salt = salt,
        iterations = 100000,
        key_length = 32,
        hash_algorithm = "SHA256"
    )
    
    // Decrypt backup blob
    try:
        backup_json = decrypt_with_key(encrypted_backup, cloud_key)
        backup_blob = json_decode(backup_json)
    except DecryptionException:
        throw InvalidPasswordException("Incorrect backup password")
    
    // Get device HSM master key
    master_key = hsm_get_master_key()
    
    // Restore accounts
    restored_count = 0
    for backup_account in backup_blob.accounts:
        // Decrypt secret with cloud key
        plaintext_secret = decrypt_with_key(
            backup_account.encrypted_secret,
            cloud_key
        )
        
        // Re-encrypt with device HSM key
        encrypted_secret = encrypt_secret(plaintext_secret, master_key)
        
        // Check if account already exists
        existing = database.find_account(
            issuer = backup_account.issuer,
            account_name = backup_account.account_name
        )
        
        if existing:
            // Update existing account
            existing.encrypted_secret = encrypted_secret
            existing.algorithm = backup_account.algorithm
            existing.digits = backup_account.digits
            existing.period = backup_account.period
            existing.icon_url = backup_account.icon_url
            database.update_account(existing)
        else:
            // Create new account
            account_record = {
                "account_id": backup_account.account_id,
                "issuer": backup_account.issuer,
                "account_name": backup_account.account_name,
                "encrypted_secret": encrypted_secret,
                "algorithm": backup_account.algorithm,
                "digits": backup_account.digits,
                "period": backup_account.period,
                "icon_url": backup_account.icon_url,
                "added_at": current_timestamp(),
                "last_used": current_timestamp()
            }
            database.insert_account(account_record)
        
        restored_count = restored_count + 1
    
    // Verify restore by generating codes
    for account in database.get_all_accounts():
        try:
            secret = decrypt_secret(account.encrypted_secret, master_key)
            code = generate_totp(secret)
            logger.info("Account restored and verified", account_id = account.account_id)
        except Exception:
            logger.warning("Account restore verification failed", account_id = account.account_id)
    
    return restored_count
```

**Time Complexity:** O(n) where n is number of accounts

**Example Usage:**

```
backup_id = backup_to_cloud("mypassword123")
// Later, on new device:
restored_count = restore_from_cloud("mypassword123", backup_id)
```

---

## Push Notification Approvals

### send_push_approval()

Sends push notification approval request to user's device.

**Parameters:**

- `user_id` (string): User ID
- `session_id` (string): Login session ID
- `ip_address` (string): IP address of login attempt
- `service_name` (string): Name of service requesting approval

**Returns:**

- `string`: Request ID

**Algorithm:**

```
function send_push_approval(user_id, session_id, ip_address, service_name):
    // Lookup user's registered devices
    devices = database.get_user_devices(user_id)
    
    if len(devices) == 0:
        throw NoDeviceException("User has no registered devices")
    
    // Generate request ID
    request_id = generate_uuid()
    
    // Create approval request
    approval_request = {
        "request_id": request_id,
        "user_id": user_id,
        "session_id": session_id,
        "ip_address": ip_address,
        "service_name": service_name,
        "timestamp": current_timestamp(),
        "expires_at": current_timestamp() + 30  // 30 second timeout
    }
    
    // Store request in database
    database.insert_approval_request(approval_request)
    
    // Send push notification to all devices
    for device in devices:
        push_token = device.push_token
        push_message = {
            "title": "Approve login?",
            "body": service_name + " is requesting access from " + ip_address,
            "data": {
                "type": "approval_request",
                "request_id": request_id,
                "session_id": session_id
            }
        }
        
        try:
            if device.platform == "ios":
                apns_send(push_token, push_message)
            else if device.platform == "android":
                fcm_send(push_token, push_message)
        except PushException:
            logger.warning("Push notification failed", device_id = device.device_id)
    
    // Schedule timeout cleanup
    schedule_timeout(request_id, 30)
    
    return request_id
```

**Time Complexity:** O(n) where n is number of devices

### handle_approval()

Handles user's approval/denial response.

**Parameters:**

- `request_id` (string): Request ID
- `action` (string): "approve" or "deny"
- `device_id` (string): Device ID that sent response

**Returns:**

- `object`: Approval result with status and session_id

**Algorithm:**

```
function handle_approval(request_id, action, device_id):
    // Get approval request
    request = database.find_approval_request(request_id)
    
    if not request:
        throw InvalidRequestException("Request not found")
    
    // Check if request expired
    if current_timestamp() > request.expires_at:
        throw ExpiredRequestException("Request expired")
    
    // Verify device is registered for user
    device = database.find_device(device_id)
    if not device or device.user_id != request.user_id:
        throw UnauthorizedException("Device not authorized")
    
    // Update request status
    request.status = action  // "approved" or "denied"
    request.responded_at = current_timestamp()
    request.device_id = device_id
    database.update_approval_request(request)
    
    // Notify website via WebSocket
    websocket_send(request.session_id, {
        "status": action,
        "request_id": request_id,
        "timestamp": current_timestamp()
    })
    
    // Cleanup expired requests
    cleanup_expired_requests()
    
    return {
        "status": action,
        "session_id": request.session_id,
        "request_id": request_id
    }
```

**Time Complexity:** O(1)

**Example Usage:**

```
request_id = send_push_approval("user-123", "session-456", "192.168.1.1", "Gmail")
// User taps "Approve" on device
result = handle_approval(request_id, "approve", "device-789")
```

---

## Time Synchronization and Drift Handling

### validate_totp_with_drift()

Validates TOTP code with time drift tolerance.

**Parameters:**

- `code` (string): User-entered TOTP code
- `secret` (string): Account secret
- `stored_offset` (integer, optional): Stored time offset in seconds

**Returns:**

- `boolean`: True if code is valid, False otherwise

**Algorithm:**

```
function validate_totp_with_drift(code, secret, stored_offset = 0):
    // Get current time
    current_time = current_unix_timestamp()
    
    // Apply stored offset
    adjusted_time = current_time + stored_offset
    
    // Try current time step
    expected_code = generate_totp(secret, adjusted_time)
    if code == expected_code:
        return true
    
    // Try previous time step (-30 seconds)
    prev_code = generate_totp(secret, adjusted_time - 30)
    if code == prev_code:
        return true
    
    // Try next time step (+30 seconds)
    next_code = generate_totp(secret, adjusted_time + 30)
    if code == next_code:
        return true
    
    // Code not valid in any time window
    return false
```

**Time Complexity:** O(1)

### sync_time_with_ntp()

Synchronizes device time with NTP server and calculates offset.

**Parameters:**

- None

**Returns:**

- `integer`: Time offset in seconds (positive = device ahead, negative = device behind)

**Algorithm:**

```
function sync_time_with_ntp():
    // Check internet connectivity
    if not is_online():
        throw NetworkException("No internet connection")
    
    // Query NTP server
    try:
        ntp_time = ntp_query("pool.ntp.org")
    except NetworkException:
        throw NetworkException("NTP query failed")
    
    // Get device time
    device_time = current_unix_timestamp()
    
    // Calculate offset
    offset = ntp_time - device_time
    
    // Store offset in database
    database.save_time_offset(offset, current_timestamp())
    
    // Log if offset is significant
    if abs(offset) > 30:
        logger.warning("Significant time drift detected", offset = offset)
    
    return offset
```

**Time Complexity:** O(1)

**Example Usage:**

```
offset = sync_time_with_ntp()
// offset = 5 means device is 5 seconds behind NTP
code = generate_totp(secret, current_time() + offset)
is_valid = validate_totp_with_drift(user_code, secret, offset)
```

