# Implementation Plan: Zero-Knowledge Password Recovery & Key Rotation

This document outlines the design and implementation strategy for adding **Zero-Knowledge Password Recovery** and **Key Rotation** capabilities to DocOps.

---

## 1. Design Overview

To support instant password changes, recovery, and key rotation without having to re-encrypt any files in storage, we will transition the key hierarchy to a **Master Key (MK) + Key Encryption Key (KEK)** model.

```
       [ User Password ]                     [ Recovery Key ]
               │                                     │
               ▼ (Argon2id)                          ▼ (Argon2id)
           [  KEK  ]                             [ KEK_recovery ]
               │                                     │
               ▼ (AES-256-GCM)                       ▼ (AES-256-GCM)
      ┌─────────────────────────────────────────────────────────────┐
      │                      [ Master Key (MK) ]                     │
      └──────────────────────────────┬──────────────────────────────┘
                                     │
                        (AES-256-GCM)│(Envelope Encryption)
                                     ▼
                        [ Document DEKs ]
                                     │
                        (AES-256-GCM)│(Stream Encryption)
                                     ▼
                            [ File Plaintext ]
```

### Key Components:
1. **Master Key (MK)**: A random 256-bit symmetric key generated at registration. It is never changed (unless explicit Master Key rotation is triggered). The MK is used to wrap/unwrap all document DEKs.
2. **Key Encryption Key (KEK)**: Derived from the user's password using Argon2id. It is used solely to encrypt/decrypt the Master Key.
3. **Recovery Key**: A high-entropy random string generated at registration (e.g. `docops_rec_xxxx...` or a mnemonic phrase). It derives a `KEK_recovery` which wraps the same Master Key.

---

## 2. Proposed Database Schema Changes

We will modify the `users` table to store the wrapped Master Key and recovery wrapping details:

```sql
ALTER TABLE users ADD COLUMN wrapped_master_key BLOB;
ALTER TABLE users ADD COLUMN master_key_nonce BLOB;
ALTER TABLE users ADD COLUMN recovery_salt BLOB;
ALTER TABLE users ADD COLUMN recovery_wrapped_master_key BLOB;
ALTER TABLE users ADD COLUMN recovery_master_key_nonce BLOB;
```

For v0.1, we will update the `migrate` function in `services/auth/users.go` to recreate the table or perform these migrations automatically.

---

## 3. Workflow Details

### A. Registration Flow (`POST /v0.1/auth/register`)
1. User provides `email` and `password`.
2. Server generates a random 32-byte **Master Key (MK)**.
3. Server generates a random 20-character **Recovery Key** (e.g., `docops_rec_` followed by base32 characters).
4. **Password Wrapping**:
   - Derive `KEK` from `password` and `salt`.
   - Wrap `MK` using `KEK` -> `wrapped_master_key`, `master_key_nonce`.
5. **Recovery Wrapping**:
   - Generate `recovery_salt`.
   - Derive `KEK_recovery` from `Recovery Key` and `recovery_salt`.
   - Wrap `MK` using `KEK_recovery` -> `recovery_wrapped_master_key`, `recovery_master_key_nonce`.
6. Save user to database.
7. Return the raw `Recovery Key` to the user (visible only once).
8. Save the decrypted `MasterKey` in the user's session (`session.KEK`).

### B. Login Flow (`POST /v0.1/auth/login`)
1. User provides `email` and `password`.
2. Fetch user record.
3. Verify password hash.
4. Derive `KEK` from `password` and `salt`.
5. Decrypt `wrapped_master_key` using `KEK` and `master_key_nonce` to obtain the `MasterKey`. If decryption fails, the password or keys are invalid.
6. Store `MasterKey` in the session in memory.

### C. Password Recovery Flow (`POST /v0.1/auth/recover`)
This flow resets a forgotten password using the offline Recovery Key:
1. User provides `email`, `recovery_key`, and `new_password`.
2. Fetch user record.
3. Derive `KEK_recovery` from `recovery_key` and `recovery_salt`.
4. Decrypt `recovery_wrapped_master_key` using `KEK_recovery` to obtain the `MasterKey`. If decryption fails, the recovery key is wrong.
5. Create new password hash and new KEK salt.
6. Derive a new `KEK` from `new_password` and the new salt.
7. Wrap the `MasterKey` with the new `KEK` -> `new_wrapped_master_key`, `new_master_key_nonce`.
8. Update the user record with the new password hash, salt, and wrapped Master Key.

---

## 4. Key Rotation Strategies

We propose supporting two distinct forms of key rotation:

### I. Password KEK Rotation (Low Overhead)
* **Trigger**: When a user changes their password from their profile settings page (authenticated session).
* **Implementation**:
  1. The user's active session already holds the decrypted `MasterKey` in RAM.
  2. The user submits their `old_password` (to verify session legitimacy) and `new_password`.
  3. Derives new `KEK` from `new_password`.
  4. Wraps the `MasterKey` with the new `KEK`.
  5. Updates `wrapped_master_key`, `master_key_nonce`, and `password_hash` in the database.
* **Performance**: Instant (1 database update, 0 document updates).

### II. Master Key Rotation (Medium Overhead)
* **Trigger**: Periodically (e.g. annually) or if a session store compromise is suspected.
* **Implementation**:
  1. Generate a new random `MasterKey` (`NewMasterKey`).
  2. Fetch all document metadata records belonging to the user.
  3. For each document:
     - Decrypt its `encryptedDEK` with the old `MasterKey` to get `DEK`.
     - Encrypt the `DEK` with the `NewMasterKey` -> `newEncryptedDEK`, `newDEKNonce`.
     - Update the document metadata in the database.
  4. Wrap `NewMasterKey` under the user's password-derived `KEK` and update `wrapped_master_key`.
  5. Since the recovery wrapped key also wraps the old Master Key, we must either:
     - Prompt the user to provide their Recovery Key so we can re-wrap the `NewMasterKey` under it.
     - **OR** automatically generate a **New Recovery Key**, wrap the `NewMasterKey` under it, and return the new Recovery Key to the user. We recommend this approach as it is simpler and ensures the recovery key is rotated as well.
* **Performance**: O(N) database operations where N is the number of documents. File storage is never touched.

---

## 5. Proposed Code Changes

### [MODIFY] [services/auth/users.go](file:///home/ernest-kyei/GolandProjects/DocOps/services/auth/users.go)
- Update `User` struct definition and SQLite migrations to support Master Key and recovery wrapping fields.
- Update `CreateUser`, `GetByEmail` query binding to handle new columns.

### [NEW] [handlers/recovery.go](file:///home/ernest-kyei/GolandProjects/DocOps/handlers/recovery.go)
- Create `Recover` handler.
- Create `ChangePassword` handler.
- Create `RotateMasterKey` handler.

### [MODIFY] [handlers/auth.go](file:///home/ernest-kyei/GolandProjects/DocOps/handlers/auth.go)
- Update `Register` to generate Master Key & Recovery Key, wrap them, and return Recovery Key in response.
- Update `Login` to decrypt the Master Key and store it in the session context.

### [MODIFY] [main.go](file:///home/ernest-kyei/GolandProjects/DocOps/main.go)
- Register routes for password change, recovery, and key rotation:
  - `POST /v0.1/auth/recover` (unauthenticated, rate-limited)
  - `POST /v0.1/auth/change-password` (authenticated)
  - `POST /v0.1/auth/rotate-master-key` (authenticated)

---

## 6. Open Questions / Design Decisions

> [!IMPORTANT]
> 1. **Do we want to generate a new Recovery Key during Master Key rotation?**
>    - *Yes (Recommended)*: It forces the user to save a fresh recovery key and ensures the old recovery key is rotated.
>    - *No*: Requires the user to enter their current Recovery Key during the rotation process to re-wrap the new Master Key under the old Recovery Key.
> 2. **Recovery Key format**: Do we want a random 20-character Base32 string (e.g. `docops_rec_A7F3...`), or a 12-word mnemonic phrase (like BIP-39)?
>    - A Base32 string is easier to implement without external dependencies.
