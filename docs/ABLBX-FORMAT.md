
# ABLBX File Format

ABLBX is the encrypted backup file format used by **Abdal LockBox**.

The file extension is:

```text
.ablbx
```

Recommended filename pattern:

```text
abdal-lockbox-backup-{yyyyMMdd-HHmmss}.ablbx
```

Example:

```text
abdal-lockbox-backup-20260620-184530.ablbx
```

## Purpose

The ABLBX format is designed for secure and portable vault export/import.

A valid `.ablbx` file must be:

* Encrypted
* Integrity protected
* Portable across devices
* Independent from Android Keystore, StrongBox, TEE, or any device-bound key
* Importable on another Android device using only the file and the Export Passphrase

A valid `.ablbx` file must never contain plaintext vault data.

## Security Model

ABLBX files are encrypted using an Export Passphrase provided by the user during export.

The Export Passphrase is never stored in the file.

The encryption key is derived as follows:

```text
Export Passphrase + exportSalt + KDF = exportKEK
```

Where:

* `exportKEK` means **Export Key Encryption Key**
* `KDF` means **Key Derivation Function**
* `exportSalt` is a random salt stored in the file header
* `exportKEK` is used only in memory and must never be stored

The encrypted payload is encrypted using:

```text
AEAD / AES-256-GCM
```

Where:

* `AEAD` means **Authenticated Encryption with Associated Data**
* `AES-256-GCM` means **Advanced Encryption Standard 256-bit Galois Counter Mode**

## Portability Requirement

ABLBX backup files must be portable.

This means:

* The file must not be encrypted with Android Keystore.
* The file must not be encrypted with StrongBox.
* The file must not depend on TEE.
* The file must not depend on the original phone.
* The file must not require any device-bound key for import.

To import the backup on another device, the user only needs:

```text
1. The .ablbx file
2. The Export Passphrase
```

## Magic Number

Every ABLBX file must start with this fixed magic number:

```text
ABLBX\r\n\x1A\n
```

Hex representation:

```text
41 42 4C 42 58 0D 0A 1A 0A
```

The magic number is fixed and must not change between normal app versions.

The magic number is used to identify the file as an Abdal LockBox backup file.

## File Format Version

The ABLBX file format version is independent from the Android app version.

Example:

```text
App Version Name: 1.50
ABLBX File Format Version: 1
```

The file format version must change only when the internal `.ablbx` file structure changes.

Do not change the file format version for:

* UI changes
* Bug fixes
* App version changes
* Performance improvements
* Text changes
* New screens

Change the file format version only when one of these changes:

* Binary layout
* Header structure
* Payload schema
* KDF metadata structure
* Encryption algorithm
* AEAD parameters
* Import parser compatibility
* Required fields inside encrypted payload

Recommended constant name in Kotlin:

```kotlin
const val ABLBX_FILE_FORMAT_VERSION = 1
```

## Binary Layout

ABLBX is a binary container format.

The file layout is:

```text
+----------------------------+
| Magic Number               |
+----------------------------+
| File Format Version        |
+----------------------------+
| Header Length              |
+----------------------------+
| Header JSON                |
+----------------------------+
| Encrypted Payload          |
+----------------------------+
```

### Layout Details

| Field               |     Size | Encoding          | Description                                 |
| ------------------- | -------: | ----------------- | ------------------------------------------- |
| Magic Number        |  9 bytes | ASCII             | Fixed value: `ABLBX\r\n\x1A\n`              |
| File Format Version |  2 bytes | UInt16 Big Endian | Current version: `1`                        |
| Header Length       |  4 bytes | UInt32 Big Endian | Length of Header JSON in bytes              |
| Header JSON         | variable | UTF-8 JSON        | Non-secret metadata required for decryption |
| Encrypted Payload   | variable | binary            | AEAD encrypted payload                      |

## Header JSON

The Header JSON contains only non-secret metadata required to decrypt and validate the encrypted payload.

Example:

```json
{
  "format": "AbdalLockBoxEncryptedVaultExport",
  "app": "Abdal LockBox",
  "fileFormatVersion": 1,
  "createdAt": "2026-06-20T18:45:30Z",
  "kdf": {
    "algorithm": "Argon2id",
    "params": {
      "memoryCost": 65536,
      "iterations": 3,
      "parallelism": 1,
      "outputLength": 32
    },
    "salt": "base64url-encoded-random-salt"
  },
  "aead": {
    "algorithm": "AES-256-GCM",
    "nonce": "base64url-encoded-nonce"
  },
  "payload": {
    "encoding": "binary",
    "compression": "none"
  }
}
```

## Header Security Rules

The Header JSON may contain:

* File format name
* File format version
* App name
* Creation time
* KDF algorithm
* KDF parameters
* Export salt
* AEAD algorithm
* AEAD nonce
* Payload encoding
* Compression mode

The Header JSON must never contain:

* Export Passphrase
* exportKEK
* Master Password
* Recovery Key
* DEK
* KEK
* indexKey
* Title
* Username
* Password
* Extra Info
* WebsiteURL
* Plaintext vault item data

## KDF

The Export Passphrase must be converted to `exportKEK` using a secure KDF.

Priority order:

```text
1. Argon2id
2. scrypt
3. PBKDF2-HMAC-SHA256 as last fallback only
```

The selected KDF algorithm and parameters must be stored in the Header JSON.

The KDF output length must be:

```text
32 bytes
```

This produces a 256-bit `exportKEK`.

## AEAD

The encrypted payload must be encrypted using AEAD.

Recommended algorithm:

```text
AES-256-GCM
```

AEAD provides:

* Confidentiality
* Integrity
* Tamper detection

If any byte of the encrypted payload or authenticated header data is changed, decryption must fail.

## Associated Authenticated Data

The following data must be used as AAD:

```text
Magic Number + File Format Version + Header JSON
```

This binds the encrypted payload to the file header.

If the header is modified after export, import must fail.

## Encrypted Payload

The encrypted payload contains the exported vault data.

The payload is encrypted as a single AEAD message.

Before encryption, the payload structure is:

```json
{
  "exportPayloadVersion": 1,
  "vaultMetadata": {
    "vaultId": "...",
    "schemaVersion": 1,
    "kdfAlgorithm": "...",
    "kdfParamsJson": "...",
    "salt": "...",
    "encryptedDek": "...",
    "encryptedDekNonce": "...",
    "encryptedIndexKey": "...",
    "encryptedIndexKeyNonce": "...",
    "recoveryEnabled": true,
    "recoveryKdfAlgorithm": "...",
    "recoveryKdfParamsJson": "...",
    "recoverySalt": "...",
    "encryptedRecoveryDek": "...",
    "encryptedRecoveryDekNonce": "...",
    "encryptedRecoveryIndexKey": "...",
    "encryptedRecoveryIndexKeyNonce": "...",
    "createdAt": "...",
    "updatedAt": "..."
  },
  "vaultItems": [
    {
      "id": "...",
      "vaultId": "...",
      "encryptedBlob": "...",
      "itemNonce": "...",
      "lookupHash": "...",
      "createdAt": "...",
      "updatedAt": "...",
      "deletedAt": null
    }
  ]
}
```

## Vault Item Data

Each vault item contains these logical fields:

```text
Title
Username
Password
Extra Info
WebsiteURL
```

These fields must never be stored directly in the ABLBX file.

They must exist only inside each item `encryptedBlob`.

The decrypted form of `encryptedBlob` is:

```json
{
  "title": "...",
  "username": "...",
  "password": "...",
  "extraInfo": "...",
  "websiteUrl": "..."
}
```

The ABLBX file must contain only encrypted versions of this data.

## Import Flow

When importing a `.ablbx` file:

1. Read and validate the magic number.
2. Read and validate the file format version.
3. Read the header length.
4. Read and parse the Header JSON.
5. Reject unsupported versions.
6. Read KDF parameters from the header.
7. Ask the user for the Export Passphrase.
8. Derive `exportKEK`.
9. Decrypt the encrypted payload using AEAD.
10. If decryption fails, stop import.
11. Validate the decrypted payload schema.
12. Import the vault data using Merge or Replace mode.
13. If importing on a new device, ask the user to create a new Master Password.
14. Re-wrap DEK and indexKey with the new Master Password KEK.
15. Generate a new Recovery Key.
16. Re-wrap DEK and indexKey with the new recoveryKEK.
17. Store the imported vault securely.

## Import On A New Device

When importing on a new Android device, no key from the old device must be required.

The new device import flow is:

```text
.ablbx file + Export Passphrase
        ↓
Derive exportKEK
        ↓
Decrypt encrypted payload
        ↓
Read imported DEK and indexKey
        ↓
Ask user for New Master Password
        ↓
Create new KEK
        ↓
Wrap DEK and indexKey with new KEK
        ↓
Generate new Recovery Key
        ↓
Wrap DEK and indexKey with new recoveryKEK
        ↓
Save imported vault
```

## Merge Mode

In Merge mode:

* Existing vault items must not be deleted.
* Imported items must receive new local IDs.
* ID conflicts must be avoided.
* Imported encrypted blobs must remain encrypted.
* If necessary, imported items may be re-encrypted using the current vault DEK.

## Replace Mode

In Replace mode:

* The current vault must not be deleted before import validation succeeds.
* The imported file must be fully decrypted and validated first.
* Only after successful validation may the current vault be replaced.
* If import fails, the existing vault must remain unchanged.

## Error Handling

Import must fail safely when:

* Magic number is invalid
* File format version is unsupported
* Header JSON is invalid
* KDF parameters are missing or invalid
* AEAD algorithm is unsupported
* Export Passphrase is wrong
* Encrypted payload was modified
* Payload schema is invalid
* Required fields are missing

Error messages must be generic.

Do not reveal whether the Export Passphrase, header, payload, or integrity check specifically failed.

Recommended error message:

```text
Import failed. The file is invalid, corrupted, or the passphrase is incorrect.
```

## Forbidden Content

A valid ABLBX file must never contain these values in plaintext:

* Master Password
* Export Passphrase
* Recovery Key
* DEK
* KEK
* exportKEK
* indexKey
* Title
* Username
* Password
* Extra Info
* WebsiteURL
* Plain JSON vault data

## Forbidden Designs

The following designs are forbidden:

* Encrypting the export file with Android Keystore
* Encrypting the export file with StrongBox
* Binding the export file to one phone
* Requiring the old phone to import the backup
* Storing the Export Passphrase inside the file
* Storing `exportKEK` inside the file
* Producing plaintext JSON export files
* Producing human-readable vault backups
* Using Base64 as encryption
* Using AES-ECB
* Using MD5
* Using SHA-1
* Using raw SHA-256 as password hashing
* Using custom cryptography
* Using hardcoded keys

## Recommended Kotlin Constants

```kotlin
object AblbxFormat {
    const val MAGIC = "ABLBX\r\n\u001A\n"
    const val FILE_EXTENSION = "ablbx"
    const val MIME_TYPE = "application/octet-stream"
    const val FORMAT_NAME = "AbdalLockBoxEncryptedVaultExport"
    const val ABLBX_FILE_FORMAT_VERSION = 1
}
```

## Versioning Policy

`ABLBX_FILE_FORMAT_VERSION` must remain stable as long as the file structure remains compatible.

Example:

```text
App versionName = 1.0
ABLBX_FILE_FORMAT_VERSION = 1

App versionName = 1.50
ABLBX_FILE_FORMAT_VERSION = 1
```

Only increment `ABLBX_FILE_FORMAT_VERSION` when the file structure or parser compatibility changes.

## Test Requirements

The implementation must include tests for:

* Valid magic number detection
* Invalid magic number rejection
* Supported file format version
* Unsupported file format version rejection
* Header length parsing
* Header JSON parsing
* Export Passphrase based decryption
* Wrong Export Passphrase rejection
* Payload tamper detection
* Header tamper detection
* Import on a new device without Android Keystore
* No plaintext Title in exported file
* No plaintext Username in exported file
* No plaintext Password in exported file
* No plaintext Extra Info in exported file
* No plaintext WebsiteURL in exported file
* No plaintext Master Password in exported file
* No plaintext Recovery Key in exported file
* No plaintext DEK, KEK, exportKEK, or indexKey in exported file

## Summary

ABLBX is a portable encrypted backup format.

It is identified by:

```text
ABLBX\r\n\x1A\n
```

It is versioned by:

```text
ABLBX_FILE_FORMAT_VERSION
```

It is encrypted by:

```text
Export Passphrase + exportSalt + KDF = exportKEK
```

It is protected by:

```text
AEAD / AES-256-GCM
```

It must be portable across devices and must never depend on Android Keystore or any hardware-bound key.

## Project Info

Project: Abdal LockBox  
Developer: Ebrahim Shafiei (EbraSha)  
GitHub: https://github.com/ebrasha  
Email: Prof.Shafiei@Gmail.com
