# Abdal LockBox Whitepaper

## Secure Offline Vault Management for Android

**Project:** Abdal LockBox  
**Document Type:** Technical Security Whitepaper  
**Status:** Final for Abdal LockBox v1.50  
**App Version:** 1.50  
**Developer:** Ebrahim Shafiei (EbraSha)  
**GitHub:** https://github.com/ebrasha  
**Email:** Prof.Shafiei@Gmail.com  

---

## 1. Executive Summary

Abdal LockBox is an Android security application designed for managing and protecting sensitive Vault data. The application focuses on local encrypted storage, Master Password based access control, Recovery Key based recovery, Android Autofill integration, and encrypted portable Vault backup through the `.ablbx` file format.

The architecture follows a local-first model. Vault data is stored on the device, encrypted before persistence, and accessed only after the Vault session is unlocked. The project separates user authentication material from Vault item encryption by using a layered key model based on a Master Password, Key Derivation Function (KDF), Key Encryption Key (KEK), Data Encryption Key (DEK), and indexKey.

The project also defines the **ABLBX Encrypted Vault Backup Format**, a portable encrypted backup container intended for Vault export/import. The `.ablbx` format is designed to remain independent from Android Keystore, StrongBox, TEE, or other device-bound key material, allowing users to move encrypted Vault backups between devices when they know the correct Export Passphrase.

This Whitepaper describes the project architecture, implemented capabilities, security model, data flow, storage model, import/export architecture, ABLBX format, OID namespace allocation, current limitations, and future roadmap.

---

## 2. Project Overview

Abdal LockBox is implemented as a Kotlin Android application using Jetpack Compose for UI, Room for local database persistence, DataStore Preferences for application preferences, Moshi for JSON serialization, Android Autofill APIs for credential fill/save flows, and cryptographic components for local encryption and key management.

The Android package namespace is:

```text
com.abdal.lockbox
```

The application version currently represented in the project is:

```text
versionCode = 150
versionName = "1.50"
```

The main application areas are:

- Vault creation
- Vault unlock and lock
- Master Password validation
- Master Password change
- Recovery Key generation
- Recovery Key verification
- Recovery Key rotation
- Encrypted Vault item storage
- Encrypted Vault settings
- Android Autofill integration
- ABLBX Vault export
- ABLBX Vault import
- Auto-lock behavior
- Local session management

Abdal LockBox is not designed as a cloud password manager in its current architecture. The normal Vault workflow is local and offline-first.

---

## 3. Problem Statement

A Vault application stores sensitive data such as service names, usernames, passwords, website URLs, application package bindings, and free-form notes. If these values are stored as plaintext or exported in a human-readable format, a local compromise can expose the user's full credential map.

The core problems addressed by Abdal LockBox are:

1. Sensitive Vault data must not be stored directly as plaintext.
2. Master Passwords must not be stored.
3. Vault encryption keys must not be derived or handled casually.
4. Exported backups must remain encrypted and portable.
5. Android Autofill must not expose credentials without Vault authorization.
6. Recovery must be possible only through explicit user-controlled recovery material.
7. Device loss or file extraction must not automatically expose Vault contents.

Abdal LockBox addresses these problems through application-layer encryption, key wrapping, authenticated encryption, explicit Vault session state, HMAC-based lookup values, Recovery Key handling, and encrypted backup export/import.

---

## 4. Design Goals

The project is designed around the following technical goals:

| Goal | Status |
|---|---|
| Local Vault storage | Implemented |
| Master Password based Vault unlock | Implemented |
| Recovery Key based recovery | Implemented |
| Recovery Key rotation | Implemented |
| Encrypted Vault item storage | Implemented |
| AEAD authenticated encryption | Implemented |
| KDF based key derivation | Implemented |
| Separate DEK and KEK model | Implemented |
| Separate indexKey for lookup hashes | Implemented |
| Android Autofill integration | Implemented |
| Auto-lock on lifecycle / inactivity / screen-off | Implemented |
| Portable encrypted `.ablbx` export | Implemented |
| Import with Merge / Replace / New Device modes | Implemented |
| Biometric unlock | Not part of the documented v1.50 implementation. |
| Android Keystore based Vault unlock | Not part of the documented v1.50 implementation. |
| SQLCipher database encryption | Not part of the documented v1.50 implementation. |
| XChaCha20-Poly1305 encryption | Not part of the documented v1.50 implementation. |

---

## 5. System Architecture

Abdal LockBox follows a layered architecture.

```text
UI Layer
  ├─ Compose screens
  ├─ ViewModels
  ├─ Dialogs
  └─ UI state models

Domain Layer
  ├─ Use cases
  ├─ Repository interfaces
  └─ Domain models

Data Layer
  ├─ Room database
  ├─ DAO interfaces
  ├─ Entity models
  └─ Repository implementations

Crypto Layer
  ├─ KDF service
  ├─ AEAD service
  ├─ Crypto engine
  ├─ Lookup hash service
  ├─ Recovery key generator
  ├─ Secret zeroization helper
  └─ ABLBX file codec

Session Layer
  ├─ Vault session manager
  ├─ In-memory DEK / indexKey state
  ├─ Vault lock state
  └─ Autofill authorization state

Android Integration Layer
  ├─ AutofillService
  ├─ Autofill authentication Activity
  ├─ Autofill save confirmation Activity
  ├─ Storage Access Framework document import/export
  ├─ Process lifecycle observer
  └─ Screen-off receiver
```

The application graph wires together database access, repositories, use cases, cryptographic services, session state, preferences, Autofill services, and import/export services.

The current local database contains two main persistent models:

```text
vault_metadata
vault_items
```

Sensitive Vault item values are not modeled as plaintext database columns. They are serialized and encrypted into an encrypted blob before storage.

---

## 6. Core Features

### 6.1 Vault Creation

The Vault creation flow creates the cryptographic material required to protect future Vault items.

The flow is:

```text
Master Password
        ↓
Password validation
        ↓
Salt generation
        ↓
KDF
        ↓
KEK generation
        ↓
DEK generation
        ↓
indexKey generation
        ↓
DEK and indexKey wrapping
        ↓
Recovery Key generation
        ↓
Recovery wrapping
        ↓
Default Vault settings encryption
        ↓
Vault metadata storage
        ↓
Session unlock
```

The DEK and indexKey are random keys and are not directly derived from the Master Password.

### 6.2 Vault Unlock

Vault unlock requires the user-provided Master Password.

The flow is:

```text
User Master Password
        ↓
Stored salt + KDF parameters
        ↓
Derived KEK
        ↓
Decrypt encrypted DEK
        ↓
Decrypt encrypted indexKey
        ↓
Store DEK and indexKey in active session memory
```

If unlock fails, the Vault remains locked.

### 6.3 Vault Lock

Vault lock clears active session material.

The lock operation clears:

```text
DEK
indexKey
Vault ID
Autofill fill authorization state
```

The lock behavior is used by direct user action, lifecycle auto-lock, inactivity-based auto-lock, and screen-off handling.

### 6.4 Master Password Change

The Master Password change process does not require re-encrypting every Vault item.

The high-level process is:

```text
Old Master Password
        ↓
Old KEK
        ↓
Unwrap current DEK and indexKey
        ↓
New Master Password
        ↓
New salt + KDF
        ↓
New KEK
        ↓
Re-wrap same DEK and same indexKey
```

This design keeps Vault item encryption stable while replacing the password-derived wrapping key.

### 6.5 Recovery Key

The Recovery Key is a user-controlled recovery secret used to restore access when the Master Password is unavailable.

The Recovery Key flow uses a separate key derivation path:

```text
Recovery Key
        ↓
Recovery salt + KDF
        ↓
recoveryKEK
        ↓
Unwrap recovery-wrapped DEK and indexKey
```

The current implementation supports:

- Recovery Key generation
- Recovery Key confirmation
- Recovery Key verification
- Vault recovery
- Recovery Key rotation

### 6.6 Vault Item Management

Each Vault item can store sensitive and operational fields such as:

```text
title
username
password
extraInfo
websiteUrl
cardColor
textColor
enableAutofill
appPackage
normalizedDomain
lastUsedAt
```

The plaintext item exists only before encryption or after decryption in memory. The persistent model stores encrypted content and lookup hashes.

### 6.7 Android Autofill

Abdal LockBox implements Android Autofill support for filling and saving credentials.

Autofill behavior includes:

- Parsing Android AssistStructure
- Detecting username and password fields
- Requiring Vault unlock before fill
- Optional Master Password reauthorization before fill
- Building Autofill datasets
- Saving detected credentials after user confirmation
- Matching by browser domain or app package
- Respecting item-level Autofill enablement

### 6.8 Import / Export

Abdal LockBox supports encrypted Vault export/import through the ABLBX format.

Export requires:

```text
Unlocked Vault
Export Passphrase
```

Import requires:

```text
.ablbx file
Export Passphrase
Selected import policy
```

Supported import modes are:

```text
MERGE
REPLACE
NEW_DEVICE
```

---

## 7. Security Model

### 7.1 Key Hierarchy

The implemented security model separates authentication secrets from data encryption keys.

```text
Master Password
        ↓
KDF + salt
        ↓
KEK
        ↓
unwraps
DEK
        ↓
decrypts
Vault item encrypted blobs
```

The lookup model uses a separate index key:

```text
indexKey
        ↓
HMAC-SHA-256
        ↓
lookup_hash / package_lookup_hash
```

Recovery uses a separate key path:

```text
Recovery Key
        ↓
KDF + recovery salt
        ↓
recoveryKEK
        ↓
unwraps
DEK and indexKey
```

Export uses a separate key path:

```text
Export Passphrase
        ↓
KDF + export salt
        ↓
exportKEK
        ↓
decrypts ABLBX encrypted payload
```

### 7.2 Master Password Handling

The Master Password is the primary authentication secret.

Current rules:

```text
Minimum length: 6 characters
Maximum length: 256 characters
```

The current validation is length-based. Composition requirements such as uppercase letters, lowercase letters, digits, symbols, or entropy scoring are not explicitly enforced as hard requirements.

The Master Password is used to derive a KEK. It is not used directly to encrypt Vault items.

### 7.3 KDF

The current cryptographic model includes KDF support for:

```text
ARGON2ID
SCRYPT
PBKDF2_HMAC_SHA256
```

Argon2id is used as the primary derivation algorithm in the implemented model.

The currently represented default Argon2id parameters are:

```text
memoryCost = 32768
iterations = 3
parallelism = 2
outputLength = 32
```

PBKDF2-HMAC-SHA256 is represented as a fallback derivation method.

The currently represented PBKDF2 parameters are:

```text
iterations = 600000
outputLength = 32
```

A distinct scrypt implementation is not part of the documented v1.50 implementation, even though scrypt is represented in the KDF model and reserved cryptographic namespace.

### 7.4 AEAD Encryption

Vault encryption uses AEAD-style authenticated encryption.

The implemented encryption algorithm is:

```text
AES-GCM
```

The expected key size is:

```text
32 bytes
```

The nonce size used by the encryption layer is:

```text
12 bytes
```

The authentication tag size is:

```text
128 bits / 16 bytes
```

XChaCha20-Poly1305 is reserved in the OID namespace but is not explicitly implemented in the documented v1.50 implementation.

### 7.5 AAD

Associated Authenticated Data is used to bind encrypted content to contextual metadata.

AAD purposes include:

```text
Vault item encryption
DEK wrapping
indexKey wrapping
Recovery DEK wrapping
Recovery indexKey wrapping
Vault settings encryption
ABLBX export payload encryption
```

Structured canonical AAD encoding is not part of the documented v1.50 implementation.

### 7.6 Lookup Hashing

Lookup values are protected through HMAC-based hashing.

The model is:

```text
lookupHash = HMAC-SHA-256(indexKey, normalizedDomain)
```

For app-based Autofill matching, package lookup hashes are also represented.

This allows indexed lookup without storing raw lookup values as direct searchable plaintext database columns.

### 7.7 Secret Zeroization

The project includes explicit zeroization for byte and character arrays.

The zeroization model applies to sensitive arrays such as:

```text
Master Password char arrays
Recovery Key char arrays
Export Passphrase char arrays
DEK byte arrays
indexKey byte arrays
KEK byte arrays
Pending Autofill password char arrays
```

However, immutable JVM `String` values cannot be reliably zeroized after creation. Any code path that converts sensitive data to `String` inherits this limitation.

---

## 8. Data Flow

### 8.1 Vault Creation Flow

```text
Master Password input
        ↓
Password validation
        ↓
Generate salt
        ↓
Derive KEK
        ↓
Generate DEK
        ↓
Generate indexKey
        ↓
Wrap DEK with KEK
        ↓
Wrap indexKey with KEK
        ↓
Generate Recovery Key
        ↓
Derive recoveryKEK
        ↓
Wrap DEK with recoveryKEK
        ↓
Wrap indexKey with recoveryKEK
        ↓
Encrypt default settings
        ↓
Persist Vault metadata
        ↓
Unlock active session
```

### 8.2 Vault Unlock Flow

```text
Master Password input
        ↓
Load Vault metadata
        ↓
Read KDF algorithm and parameters
        ↓
Derive KEK
        ↓
Decrypt encrypted DEK
        ↓
Decrypt encrypted indexKey
        ↓
Store keys in VaultSessionManager
```

### 8.3 Vault Item Save Flow

```text
Plaintext Vault item
        ↓
Normalize website URL / app package
        ↓
Generate lookup hashes using indexKey
        ↓
Serialize Vault item
        ↓
Encrypt serialized item using DEK + AEAD
        ↓
Store encrypted blob and metadata in Room
```

### 8.4 Vault Item Read Flow

```text
Read encrypted VaultItemEntity
        ↓
Load DEK from active session
        ↓
Decrypt encrypted_blob
        ↓
Deserialize plaintext Vault item
        ↓
Return domain model to UI / Autofill
```

### 8.5 Autofill Fill Flow

```text
Android FillRequest
        ↓
Parse AssistStructure
        ↓
Detect username/password fields
        ↓
Check Autofill settings
        ↓
Require Vault unlock if locked
        ↓
Optionally require fill reauthorization
        ↓
Normalize domain or package
        ↓
Search by lookup hash
        ↓
Decrypt matching Vault items
        ↓
Build FillResponse datasets
```

### 8.6 Autofill Save Flow

```text
Android SaveRequest
        ↓
Parse username/password values
        ↓
Store pending save payload in short-lived memory
        ↓
Require Vault unlock if needed
        ↓
Show save confirmation UI
        ↓
Create encrypted Vault item
```

### 8.7 Export Flow

```text
Unlocked Vault
        ↓
Read Vault metadata
        ↓
Read active Vault items
        ↓
Read encrypted settings
        ↓
Build ABLBX payload
        ↓
Read Export Passphrase
        ↓
Derive exportKEK
        ↓
Encrypt payload
        ↓
Write ABLBX binary file
```

### 8.8 Import Flow

```text
Read ABLBX file
        ↓
Validate magic number
        ↓
Validate format version
        ↓
Parse header
        ↓
Read Export Passphrase
        ↓
Derive exportKEK
        ↓
Decrypt payload
        ↓
Validate payload
        ↓
Apply MERGE / REPLACE / NEW_DEVICE policy
```

---

## 9. Vault Data Model

### 9.1 Logical Vault Entry

A Vault entry contains user-facing credential data and optional metadata.

```text
Title
Username
Password
Extra Info
Website URL
Card Color
Text Color
Enable Autofill
App Package
Normalized Domain
Last Used At
```

### 9.2 Persistent Vault Entry

The persistent Vault item entity stores encrypted content and lookup metadata.

```text
id
vault_id
encrypted_blob
item_nonce
lookup_hash
package_lookup_hash
created_at
updated_at
deleted_at
```

The following sensitive logical fields are not stored as dedicated plaintext database columns:

```text
title
username
password
extraInfo
websiteUrl
```

### 9.3 Vault Metadata

Vault metadata includes:

```text
vault_id
schema_version
kdf_algorithm
kdf_params_json
salt
encrypted_dek
encrypted_dek_nonce
encrypted_index_key
encrypted_index_key_nonce
recovery_enabled
recovery_kdf_algorithm
recovery_kdf_params_json
recovery_salt
encrypted_recovery_dek
encrypted_recovery_dek_nonce
encrypted_recovery_index_key
encrypted_recovery_index_key_nonce
recovery_key_created_at
recovery_key_rotated_at
encrypted_settings_blob
settings_nonce
created_at
updated_at
```

This metadata allows the Vault to be unlocked, recovered, and configured without storing raw DEK, raw indexKey, Master Password, or Recovery Key.

---

## 10. Storage Model

### 10.1 Room Database

The application uses Room for local persistence.

Database name:

```text
lockbox_vault.db
```

Main entities:

```text
VaultMetadataEntity
VaultItemEntity
```

The documented v1.50 model uses application-layer encryption for Vault items. Full database encryption through SQLCipher is not part of the documented v1.50 implementation.

### 10.2 DataStore Preferences

Application preferences are stored with Android DataStore Preferences.

The preference model includes auto-lock related values such as:

```text
auto_lock_seconds
last_user_activity_at
```

These preferences are used for inactivity-based Vault locking.

### 10.3 Android Backup

The Android application manifest disables Android backup:

```xml
android:allowBackup="false"
```

Detailed manual exclusions inside backup rule XML files are not part of the documented v1.50 implementation beyond the manifest-level backup disablement.

---

## 11. Import / Export Architecture

### 11.1 Export Policy

Export requires:

```text
Unlocked Vault session
Export Passphrase
Writable output document
```

Export creates an encrypted `.ablbx` backup file.

The default filename pattern is:

```text
abdal-lockbox-backup-{yyyyMMdd-HHmmss}.ablbx
```

The Export Passphrase is used only for deriving the exportKEK. It is not stored inside the backup file.

### 11.2 Import Policy

Import requires:

```text
ABLBX file
Export Passphrase
Import mode
```

Supported import modes:

```text
MERGE
REPLACE
NEW_DEVICE
```

Import must validate the container structure, derive the exportKEK, decrypt the encrypted payload, validate the payload schema, and then apply the selected import policy.

### 11.3 Merge Mode

Merge mode imports data into an existing unlocked Vault.

Expected behavior:

- Existing Vault items remain present.
- Imported items are decrypted with the imported DEK.
- Imported items are re-encrypted using the current Vault session DEK.
- New local item IDs are assigned.
- Current Vault metadata remains the primary local Vault metadata.

### 11.4 Replace Mode

Replace mode replaces the local Vault data with imported data.

Expected behavior:

- Current Vault access is verified.
- Imported payload is decrypted.
- Imported Vault key material is re-wrapped for the current local Master Password.
- Local Vault metadata and items are updated.

A transaction-safe replace flow should validate and stage all imported data before deleting existing data. Full transaction safety should be treated as an important hardening requirement.

### 11.5 New Device Mode

New Device mode is intended for restoring an ABLBX backup onto a new local Vault context.

Expected behavior:

- The imported payload is decrypted using the Export Passphrase.
- A new Master Password is provided.
- The imported DEK and indexKey are wrapped with the new KEK.
- A new Recovery Key is generated.
- Imported Vault items are restored under the new local Vault identity.

### 11.6 SAF Document Provider Usage

The import/export UI uses Android document picker style operations.

Export uses document creation semantics.

Import uses document open semantics.

This corresponds to Android Storage Access Framework based file selection and creation.

---

## 12. ABLBX File Format

ABLBX is the encrypted backup file format used by Abdal LockBox.

### 12.1 Format Identifier

```text
ABLBX Encrypted Vault Backup Format
```

### 12.2 Format OID

```text
1.3.6.1.4.1.66033.1.2.2.1
```

### 12.3 Magic Number

Hex:

```text
41 42 4C 42 58 0D 0A 1A 0A
```

Text form:

```text
ABLBX\r\n\x1A\n
```

### 12.4 File Extension

```text
.ablbx
```

### 12.5 Media Type

```text
application/vnd.abdalsecuritygroup.lockbox
```

### 12.6 Format Version

The current format version model is:

```text
FORMAT_MAJOR_VERSION = 1
FORMAT_MINOR_VERSION = 0
```

The binary layout stores major and minor format version bytes.

### 12.7 Header Schema

The ABLBX header contains non-secret metadata required to parse and decrypt the backup.

Header fields include:

```text
formatName
formatOid
mediaType
formatVersion
appName
createdAt
kdfAlgorithm
kdfParams
exportSalt
encryptionAlgorithm
exportNonce
payloadLength
authenticationTagLength
headerLength
flags
```

### 12.8 Encrypted Payload Schema

The encrypted payload contains the exported Vault material.

Payload fields include:

```text
exportSchemaVersion
vaultId
dek
indexKey
vaultSettings
vaultItems
```

Each exported Vault item includes:

```text
id
encryptedBlob
itemNonce
lookupHash
packageLookupHash
createdAt
updatedAt
```

The DEK and indexKey are inside the encrypted payload, not in the cleartext header.

### 12.9 Binary Layout

The current binary container layout is:

```text
Magic Number
Major Version byte
Minor Version byte
Header Length - 4 bytes
Header JSON
Ciphertext body
Authentication tag
```

### 12.10 Integrity Metadata

The authentication tag length is represented in the header.

The ABLBX AEAD authentication covers the encrypted payload and associated metadata. AAD is built from the container prefix and header material.

If the header, ciphertext, or authentication tag is modified, decryption is expected to fail.

### 12.11 Backup Filename Pattern

```text
abdal-lockbox-backup-{yyyyMMdd-HHmmss}.ablbx
```

### 12.12 Import Policy

ABLBX import validates:

```text
Magic number
Format version
Header length
Header JSON
Format OID
Format name
Media type
Encryption algorithm
Salt
Nonce
Authentication tag length
Encrypted payload
Payload schema version
Vault ID
```

### 12.13 Export Policy

ABLBX export requires an unlocked Vault and a user-provided Export Passphrase.

The Export Passphrase is used to derive:

```text
exportKEK
```

The exportKEK encrypts the ABLBX payload.

### 12.14 SAF Document Provider Usage

ABLBX files are exported and imported through Android document provider flows.

The current media type used by the project is:

```text
application/vnd.abdalsecuritygroup.lockbox
```

---

## 13. Cryptography / Authentication / Authorization

### 13.1 Implemented Cryptographic Components

| Component | Status |
|---|---|
| Master Password handling | Implemented |
| KDF | Implemented |
| Argon2id | Implemented |
| PBKDF2-HMAC-SHA256 fallback | Implemented |
| scrypt | Declared / reserved, but not explicitly implemented as a distinct derivation path |
| AES-GCM | Implemented |
| XChaCha20-Poly1305 | Reserved / not explicitly implemented |
| DEK | Implemented |
| KEK | Implemented |
| Export KEK | Implemented |
| Recovery KEK | Implemented |
| Salt generation and storage | Implemented |
| Nonce generation | Implemented |
| Authentication tag validation | Implemented through AEAD |
| Local encryption | Implemented |
| Vault locking behavior | Implemented |
| Auto Lock Timeout | Implemented |
| Recovery Key | Implemented |
| Import / Export encryption | Implemented |
| Android SAF usage | Implemented through document flows |
| Secure storage mechanisms | Application-layer encrypted blobs implemented |
| Backup and restore behavior | Manifest backup disabled; ABLBX export/import implemented |

### 13.2 Authentication Model

The primary authentication factor is the Master Password.

The application does not currently define biometric unlock.

The Autofill authorization model is session-based:

```text
Vault locked
        ↓
Autofill requires unlock

Vault unlocked
        ↓
Autofill may provide suggestions

Require Master Password Before Fill enabled
        ↓
Autofill requires fill reauthorization
```

### 13.3 Authorization Model

The effective authorization boundary is the Vault session.

When locked:

```text
DEK unavailable
indexKey unavailable
Vault item decryption unavailable
Autofill fill unavailable
```

When unlocked:

```text
DEK available in memory
indexKey available in memory
Vault item decryption available
Autofill matching available
```

### 13.4 Recovery Authorization

Recovery requires the correct Recovery Key. A successful Recovery Key verification unwraps the Vault keys and allows the user to establish a new Master Password.

### 13.5 Export Authorization

Export requires the Vault to be unlocked and requires a separate Export Passphrase for encrypting the exported backup.

---

## 14. Privacy Considerations

Abdal LockBox minimizes direct plaintext persistence of sensitive Vault data by storing user credential fields inside encrypted blobs.

Sensitive fields include:

```text
title
username
password
extraInfo
websiteUrl
```

The current design uses lookup hashes for domain and package matching instead of storing direct lookup identifiers as primary searchable fields.

The app disables Android backup at the manifest level.

Privacy-relevant metadata still exists in the system. Examples include timestamps, encrypted item existence, item count, update time, deleted-at state, lookup hash presence, and Autofill activity metadata.

Autofill activity metadata may include:

```text
timestamp
action
title
sourceLabel
```

Even when classified as non-sensitive operational metadata, titles and source labels may reveal usage patterns. This should be considered when assessing privacy exposure.

The documented v1.50 implementation does not explicitly show Vault item data being sent to a remote server. Cloud synchronization is not part of the documented v1.50 implementation.

---

## 15. Threat Model

### 15.1 Threats Addressed

| Threat | Mitigation |
|---|---|
| Local database extraction | Sensitive Vault item fields are stored in encrypted blobs |
| Master Password disclosure from storage | Master Password is not stored |
| Raw DEK exposure from database | DEK is stored wrapped |
| Raw indexKey exposure from database | indexKey is stored wrapped |
| Backup file theft | ABLBX payload is encrypted with Export Passphrase derived key |
| Wrong Export Passphrase | AEAD decryption fails |
| Vault item tampering | AEAD authentication tag validation |
| Domain lookup disclosure | HMAC lookup hash model |
| App going to background | Auto-lock lifecycle behavior |
| Screen-off exposure | Screen-off lock receiver |
| Autofill access while locked | Autofill requires Vault unlock |

### 15.2 Threats Partially Addressed

| Threat | Notes |
|---|---|
| Metadata leakage | Some non-secret metadata remains outside encrypted blobs |
| Memory extraction | ByteArray and CharArray zeroization exists, but JVM Strings cannot be reliably wiped |
| Autofill phishing | Domain and package matching exist, but certificate hash binding is not explicitly defined |
| Local full database theft | Item contents are encrypted, but full database encryption is not explicitly defined |
| Import replace failure | Full atomic replace behavior should be hardened |

### 15.3 Threats Not Explicitly Addressed

```text
Root detection
Debugger detection
Emulator detection
Runtime hooking detection
Screenshot protection / FLAG_SECURE
Package signing certificate based Autofill binding
Hardware-backed key protection
StrongBox key protection
Android Keystore based Vault unlock
Cloud synchronization compromise
Remote account takeover
```

---

## 16. OID Registry / Namespace Allocation

Abdal LockBox defines an OID namespace for formally identifying project components, file formats, configuration keys, cryptographic parameters, Vault data model elements, recovery system identifiers, and import/export operations.

The root OID is:

```text
1.3.6.1.4.1.66033 Abdal
```

The final OID for the ABLBX Encrypted Vault Backup Format is:

```text
1.3.6.1.4.1.66033.1.2.2.1
```

This namespace is used to identify official technical elements of Abdal LockBox. Some branches are implemented in the documented v1.50 implementation, while others are reserved as design-level namespaces for future stability and documentation.

### 16.1 Abdal LockBox OID Tree

```text
1.3.6.1.4.1.66033 Abdal
 ├─ .0 Organization Metadata
 ├─ .1 Software Products
 │
 │   └─ .2 Abdal LockBox
 │       ├─ .1 Product Info
 │       │   ├─ .1 Product Name
 │       │   ├─ .2 Product Version
 │       │   ├─ .3 Package Name
 │       │   ├─ .4 Build Number
 │       │   └─ .5 Release Channel
 │       │
 │       ├─ .2 File Formats
 │       │   └─ .1 ABLBX Encrypted Vault Backup Format
 │       │       ├─ .1 Format Identifier
 │       │       ├─ .2 Magic Number
 │       │       ├─ .3 File Extension
 │       │       ├─ .4 Media Type
 │       │       ├─ .5 Format Version
 │       │       ├─ .6 Header Schema
 │       │       ├─ .7 Encrypted Payload Schema
 │       │       └─ .8 Integrity Metadata
 │       │
 │       ├─ .3 Config Keys
 │       │   ├─ .1 Auto Lock Timeout
 │       │   ├─ .2 Theme Mode
 │       │   ├─ .3 Vault Lock Policy
 │       │   ├─ .4 Export Policy
 │       │   └─ .5 Import Policy
 │       │
 │       ├─ .4 Cryptographic Parameters
 │       │   ├─ .1 KDF Parameters
 │       │   │   ├─ .1 Argon2id
 │       │   │   └─ .2 scrypt
 │       │   ├─ .2 Encryption Algorithms
 │       │   │   ├─ .1 AES-256-GCM
 │       │   │   └─ .2 XChaCha20-Poly1305
 │       │   ├─ .3 Key Types
 │       │   │   ├─ .1 DEK
 │       │   │   ├─ .2 KEK
 │       │   │   └─ .3 Export KEK
 │       │   ├─ .4 Nonce Policy
 │       │   ├─ .5 Salt Policy
 │       │   └─ .6 Authentication Tag Policy
 │       │
 │       ├─ .5 Vault Data Model
 │       │   ├─ .1 Vault Entry
 │       │   ├─ .2 Username Field
 │       │   ├─ .3 Password Field
 │       │   ├─ .4 Website URL Field
 │       │   ├─ .5 Extra Info Field
 │       │   └─ .6 Index Key
 │       │
 │       ├─ .6 Recovery System
 │       │   ├─ .1 Recovery Key Format
 │       │   ├─ .2 Recovery Verification
 │       │   └─ .3 Recovery Policy
 │       │
 │       └─ .7 Import Export Operations
 │           ├─ .1 Export Vault
 │           ├─ .2 Import Vault
 │           ├─ .3 Create Recovery Key
 │           ├─ .4 Backup Filename Pattern
 │           └─ .5 SAF Document Provider Usage
```

### 16.2 OID Implementation Status

| OID Area | Status |
|---|---|
| `1.3.6.1.4.1.66033.1.2.2.1` ABLBX format OID | Implemented |
| Organization Metadata | Reserved / Design-level Namespace |
| Product Info | Reserved / Design-level Namespace |
| File Format metadata | Partially implemented |
| Config Keys | Reserved / Design-level Namespace |
| Cryptographic Parameters | Reserved / Design-level Namespace |
| Vault Data Model | Reserved / Design-level Namespace |
| Recovery System | Reserved / Design-level Namespace |
| Import Export Operations | Reserved / Design-level Namespace |

The OID namespace provides a stable identification structure for future documentation, interoperability, metadata tagging, and formal technical references.

---

## 17. Technical Implementation Details

### 17.1 Platform

```text
Language: Kotlin
UI Framework: Jetpack Compose
Database: Room
Preferences: Android DataStore Preferences
Serialization: Moshi
Crypto Libraries / APIs: Java Cryptography Architecture, Google Tink, Argon2Kt
Min SDK: 26
Target SDK: 36
Application ID: com.abdal.lockbox
```

### 17.2 Main Technical Modules

```text
autofill
bootstrap
crypto
crypto.ablbx
crypto.model
data.local
data.local.dao
data.local.entity
data.local.migration
data.model
data.repository
domain.model
domain.repository
domain.usecase
session
ui
ui.components
ui.screen
ui.viewmodel
util
```

### 17.3 Crypto Module

The crypto module contains:

```text
AadBuilder
AeadService
CryptoEngine
KdfService
LookupHashService
PasswordValidator
RecoveryKeyGenerator
SecretBuffer
VaultCryptoConstants
AblbxConstants
AblbxFileCodec
AblbxPayloadModels
```

### 17.4 Data Module

The data module contains:

```text
LockBoxDatabase
VaultItemDao
VaultMetadataDao
VaultItemEntity
VaultMetadataEntity
VaultItemRepositoryImpl
VaultRepositoryImpl
VaultItemPlaintext
VaultSettingsPlaintext
```

### 17.5 Session Module

The session module contains:

```text
VaultSessionManager
AppPreferences
AutoLockOption
ScreenOffLockReceiver
```

### 17.6 Autofill Module

The Autofill module contains:

```text
AbdalAutofillService
AutofillActivityLogger
AutofillAuthActivity
AutofillConfirmSaveActivity
AutofillFillResponseBuilder
AutofillMatchingEngine
AutofillSaveParser
AutofillSessionStore
AutofillStructureParser
KnownAutofillFieldMappings
```

### 17.7 Auto-Lock Options

The current auto-lock options are:

```text
Never
Immediately
1 minute
5 minutes
15 minutes
30 minutes
```

The default auto-lock behavior is represented as one minute.

### 17.8 Release Hardening

The project includes R8 / ProGuard rules for preserving required runtime components and stripping verbose, debug, and info Android log calls in release builds.

### 17.9 Release Build Hardening

Release builds are intended to use R8 / ProGuard rules for preserving required runtime components and reducing unnecessary runtime metadata. The documented release hardening also includes removal of verbose, debug, and info Android log calls where configured.

---

## 18. Limitations

The following limitations are visible from the documented v1.50 implementation:

1. **No SQLCipher full database encryption is explicitly defined.**  
   Vault item content is encrypted at the application layer, but the database file itself is not explicitly protected through SQLCipher.

2. **No Android Keystore Vault unlock is explicitly defined.**  
   The current security model is Master Password / Recovery Key based.

3. **No StrongBox integration is explicitly defined.**  
   StrongBox-backed key protection is not implemented in the documented v1.50 implementation.

4. **No biometric unlock is explicitly defined.**  
   The current UI language also indicates no Face ID or Fingerprint.

5. **scrypt is reserved but not clearly implemented as a distinct KDF path.**  
   The model includes scrypt, but an actual scrypt derivation implementation is not part of the documented v1.50 implementation.

6. **XChaCha20-Poly1305 is reserved but not implemented.**  
   The OID namespace reserves it, but the active encryption model uses AES-GCM.

7. **Autofill package certificate hash binding is not explicitly defined.**  
   Matching is based on domain and package identifiers, not explicit signing certificate validation.

8. **Structured AAD encoding is not explicitly defined.**  
   Future versions should use canonical structured AAD encoding to avoid ambiguity.

9. **Full transaction-safe Replace import should be hardened.**  
   Replace import should validate and stage all imported content before deleting or replacing existing Vault content.

10. **No explicit root, debugger, emulator, or runtime hooking detection is defined.**

11. **No explicit screenshot protection through `FLAG_SECURE` is defined.**

12. **Sensitive JVM Strings cannot be reliably zeroized.**  
   ByteArray and CharArray zeroization exists, but immutable String values remain a platform limitation.

13. **Cloud synchronization is not implemented.**  
   The current model is local-first.

---

## 19. Future Roadmap

The following roadmap items align with the current architecture:

### 19.1 Transaction-Safe Import Replace

Replace import should be implemented as an atomic operation. The application should fully parse, decrypt, validate, and stage the imported Vault before modifying the current Vault.

### 19.2 Complete scrypt Implementation

The existing scrypt namespace and parameter model should be backed by a real scrypt derivation implementation if scrypt remains part of the supported KDF list.

### 19.3 Optional SQLCipher Layer

Full-database encryption can be added as a second layer beneath encrypted Vault item blobs.

### 19.4 Optional Android Keystore Local Protection

Android Keystore can be added as an optional local protection layer for device-bound convenience features, while keeping ABLBX backups portable and independent from hardware-backed keys.

### 19.5 Stronger Autofill App Binding

Autofill matching can be hardened with package signing certificate hash verification.

### 19.6 Structured AAD Encoding

AAD should use canonical structured encoding instead of simple concatenation.

### 19.7 Explicit Backup Exclusion Rules

Even though manifest-level backup disablement exists, explicit backup and data extraction exclusion rules would provide defense-in-depth.

### 19.8 Security Test Suite

A dedicated security test suite should validate:

```text
Wrong Master Password behavior
Wrong Recovery Key behavior
Wrong Export Passphrase behavior
Header tamper detection
Payload tamper detection
Nonce uniqueness
No plaintext sensitive fields in database
No plaintext sensitive fields in ABLBX files
Recovery Key rotation invalidation
Import Merge behavior
Import Replace behavior
New Device import behavior
```

### 19.9 OID Registry Documentation

The OID namespace should be documented in a standalone registry file and referenced from the ABLBX format specification.

### 19.10 ABLBX Format Versioning Policy

The ABLBX format version should remain independent from the Android app version. It should change only when binary layout, header schema, encrypted payload schema, parser compatibility, or cryptographic format compatibility changes.

---

## 20. Conclusion

Abdal LockBox is a local-first Android Vault application built around encrypted storage, explicit session state, Master Password based key derivation, Recovery Key based recovery, HMAC-based lookup, Android Autofill integration, and encrypted portable backup.

The current architecture separates authentication from encryption by deriving KEK material from the Master Password while using random DEK and indexKey values for Vault data protection and lookup. Vault items are stored as encrypted blobs, and imported/exported Vault data is protected through the ABLBX encrypted container format.

The ABLBX format provides a formal structure for encrypted backup portability. It includes a magic number, media type, format versioning, OID, header metadata, encrypted payload schema, authentication tag metadata, and Export Passphrase based encryption.

The project also defines a broader OID namespace for identifying product metadata, file formats, configuration keys, cryptographic parameters, Vault data model elements, recovery system elements, and import/export operations. Only the ABLBX final format OID is currently implemented directly; the rest of the namespace should be treated as Reserved / Design-level Namespace unless implemented later.

The current implementation provides a strong foundation for a secure offline Vault manager. Future hardening should focus on transactional import safety, complete scrypt support if retained, optional full-database encryption, stronger Autofill binding, structured AAD encoding, explicit backup exclusion rules, and broader security testing.
