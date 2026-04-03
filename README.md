# APKv File Format Specification

Version: 1  
Status: Stable  
License: [Public Domain (Creative Commons Zero v1.0 Universal)](https://creativecommons.org/publicdomain/zero/1.0/)

---

## 1. Overview

APKv is a container format for archiving and distributing Android application packages. It encapsulates one or more APK split files together with a structured manifest, an application icon, and an optional encryption layer. The format is designed to be self-describing, transport-friendly, and protected through password-based encryption.

APKv is an independent format specification. It does not depend on any particular installer application.

---

## 2. File Extension and MIME Type

| Property   | Value                       |
|------------|-----------------------------|
| Extension  | `.apkv`                     |
| MIME type  | `application/vnd.apkv`      |

---

## 3. Container Structure

An APKv file is a ZIP archive (PKWARE ZIP, ZIP64 compatible). All entries reside at the root level of the archive with no directory nesting.

### 3.1 Plain (Unencrypted) Layout

| Entry           | Type   | Required | Description                            |
|-----------------|--------|----------|----------------------------------------|
| `manifest.json` | Text   | Yes      | Package metadata in JSON              |
| `icon.webp`     | Binary | No       | Application icon in WebP format       |
| `*.apk`         | Binary | Yes      | One or more APK split files           |

### 3.2 Encrypted Layout

| Entry           | Type   | Required | Description                                        |
|-----------------|--------|----------|----------------------------------------------------|
| `.apkv_enc`     | Marker | Yes      | Zero-byte sentinel indicating encryption is present |
| `header.json`   | Text   | Yes      | Plaintext summary for display without decryption  |
| `manifest.enc`  | Binary | Yes      | Encrypted manifest (see Section 5)                |
| `icon.enc`      | Binary | No       | Encrypted icon (see Section 5)                    |
| `payload.enc`   | Binary | Yes      | Encrypted ZIP containing all APK split files      |

The `.apkv_enc` sentinel entry must appear before all other entries to allow streaming readers to detect encryption without scanning the entire archive.

---

## 4. Entry Specifications

### 4.1 manifest.json

A UTF-8 encoded JSON object with the following fields:

| Field           | Type             | Required | Description                                            |
|-----------------|------------------|----------|--------------------------------------------------------|
| `format`        | string           | Yes      | Literal value `"apkv"`                                |
| `formatVersion` | integer          | Yes      | Format version number. Current value: `1`             |
| `packageName`   | string           | Yes      | Android package name (e.g. `com.example.app`)         |
| `versionName`   | string           | Yes      | Human-readable version string (e.g. `2.1.0`)          |
| `versionCode`   | integer          | Yes      | Numeric version code as defined in the Android manifest |
| `label`         | string           | Yes      | Localized application display name                    |
| `isSplit`       | boolean          | Yes      | `true` if the application uses split APK delivery     |
| `splits`        | array            | Yes      | Array of APK filenames present in the archive         |
| `encrypted`     | boolean          | Yes      | `true` if the archive payload is encrypted            |
| `hasIcon`       | boolean          | Yes      | `true` if an icon entry is present in the archive     |
| `exportedAt`    | integer (64-bit) | Yes      | Unix epoch timestamp in milliseconds at export time   |

Example:

```json
{
  "format": "apkv",
  "formatVersion": 1,
  "packageName": "com.example.app",
  "versionName": "2.1.0",
  "versionCode": 210,
  "label": "Example App",
  "isSplit": true,
  "splits": ["base.apk", "split_config.arm64_v8a.apk"],
  "encrypted": false,
  "hasIcon": true,
  "exportedAt": 1743667200000
}
```

### 4.2 header.json

Present only in encrypted archives. A UTF-8 encoded JSON object that provides minimal metadata readable without a password. This allows a host application to display the application name, package, and version while prompting the user for a password.

| Field           | Type    | Required | Description                                          |
|-----------------|---------|----------|------------------------------------------------------|
| `packageName`   | string  | Yes      | Android package name                                 |
| `versionName`   | string  | Yes      | Human-readable version string                        |
| `label`         | string  | Yes      | Localized application display name                   |
| `encrypted`     | boolean | Yes      | Always `true` in this context                        |
| `hasIcon`       | boolean | Yes      | `true` if `icon.enc` is present                     |

### 4.3 icon.webp

A WebP-encoded image representing the application launcher icon. The image must be square and is recommended to be encoded at 192x192 pixels using lossy compression at quality 80. This size provides sufficient visual fidelity for display purposes while keeping file size minimal.

Implementations may choose not to include this entry. The `hasIcon` field in `manifest.json` or `header.json` indicates whether the entry is present.

### 4.4 APK Split Files (plain archives)

Each `.apk` file entry corresponds to a standard Android APK file. Filenames must match the entries listed in the `splits` array of `manifest.json`. The base APK is typically named `base.apk` or uses the original filename from the source installation.

---

## 5. Encryption

### 5.1 Algorithm Summary

| Primitive  | Algorithm                                       |
|------------|-------------------------------------------------|
| Cipher     | AES-256-CBC (PKCS5Padding)                      |
| KDF        | PBKDF2-HMAC-SHA256                              |
| Iterations | 120,000                                         |
| Key length | 256 bits                                        |
| Salt       | 16 bytes, cryptographically random per entry    |
| IV / Nonce | 16 bytes, cryptographically random per entry    |

**Important:** The cipher mode is AES-CBC with PKCS#5 padding. Implementations must not substitute an AEAD mode such as AES-GCM, even though GCM is generally considered stronger. Using a different mode produces a binary layout that is incompatible with all other conforming implementations. See Section 10.5 for the rationale behind this choice.

### 5.2 Encrypted Blob Layout

Each independently encrypted entry uses the following binary layout:

```text
[ salt (16 bytes) ][ iv (16 bytes) ][ AES-CBC ciphertext ]
```

The total structural overhead per encrypted entry is 32 bytes (Salt + IV) preceding the ciphertext. The ciphertext length will be padded to a multiple of the 16-byte block size according to PKCS#5 padding rules.

**Critical implementation note:** When reading the salt and IV from a byte stream, implementations must use a blocking read that guarantees all requested bytes are consumed before proceeding. On many platforms and I/O layers (particularly Android's `InputStream` and Python's file objects in buffered mode), a single `read(n)` call is not guaranteed to return exactly `n` bytes even when more data is available. If a partial read occurs, the derived key and IV will be wrong and decryption will silently fail or produce corrupt output. Use the appropriate primitive for your platform:

| Platform / Language | Correct primitive                                     |
|---------------------|-------------------------------------------------------|
| Java / Kotlin       | `DataInputStream.readFully(buf)`                      |
| Python              | `file.read(n)` on a real file object is safe; for sockets or pipes use `socket.recv_exactly` or loop until full |
| C / C++             | Loop `read()` until all bytes received                |
| Go                  | `io.ReadFull(r, buf)`                                 |
| Rust                | `Read::read_exact(&mut buf)`                          |

### 5.3 Key Derivation

A unique key is derived for each encrypted entry independently using the same password but a freshly generated random salt. This means `manifest.enc`, `icon.enc`, and `payload.enc` each use a distinct derived key, even when protected by the same password.

```text
key = PBKDF2-HMAC-SHA256(password, salt, iterations=120000, keylen=256)
```

The password is encoded as UTF-8 before being passed to PBKDF2. Implementations must not use any other character encoding.

### 5.4 Decryption (Confidentiality)

AES-CBC provides confidentiality. Unlike AEAD modes, CBC does not inherently provide cryptographic integrity validation. Implementations will typically detect an incorrect password via a padding validation failure (e.g., `BadPaddingException`) during the final block decryption, or by validating the resulting plaintext structure (such as checking for valid JSON parsing).

**Note on `CipherInputStream` (Java / Android):** Android's `CipherInputStream` wrapping an AES-CBC cipher does not always propagate `BadPaddingException` to the caller. A failed decryption may silently produce corrupt bytes rather than throwing an exception, causing the error to manifest only when the resulting data is parsed (e.g., as JSON or as a ZIP stream). Implementations must therefore validate the decrypted plaintext structure explicitly and must not assume that a non-throwing decrypt means success.

### 5.5 Payload Structure

The `payload.enc` entry, once decrypted, yields a ZIP archive whose entries are the raw APK split files. This inner ZIP is structurally identical to the split entries in a plain archive.

---

## 6. Encryption Detection

A reader determines whether an APKv archive is encrypted by checking for the presence of the `.apkv_enc` entry. Readers must not rely on the `encrypted` field of `header.json` alone, as the sentinel entry provides a reliable structural signal without requiring JSON parsing.

---

## 7. Password Verification

To verify a password without fully decrypting the payload, a reader attempts to decrypt `manifest.enc`. If AES-CBC decryption completes (valid padding) and the resulting plaintext parses as valid JSON containing the `format` field with value `"apkv"`, the password is considered correct. This avoids the cost of decrypting `payload.enc` (which may be substantially larger) during a verification step.

---

## 8. Reading the Icon Without Installing

Both plain and encrypted archives allow the icon to be retrieved independently of the APK payload.

- In a plain archive, read the `icon.webp` entry directly.
- In an encrypted archive, decrypt the `icon.enc` entry using the verified password.

This allows host applications to display the application icon at the password prompt screen, using only the verified password and without extracting any APK content.

---

## 9. Format Versioning

The `formatVersion` field is a monotonically increasing integer. Version `1` is the baseline defined by this document. Future versions may introduce additional entries or manifest fields. Readers encountering an unknown `formatVersion` value should treat the file as potentially incompatible and inform the user accordingly, but must not refuse to attempt a read.

---

## 10. Security Considerations

### 10.1 Password Strength

The KDF iteration count of 120,000 is chosen to impose a meaningful computational cost on brute-force attacks while remaining acceptable on mobile hardware. Users should be advised that the security of an encrypted APKv archive is bounded by the entropy of the chosen password.

### 10.2 No Key Derivation Reuse

Each encrypted entry uses a freshly generated random salt, ensuring that identical passwords do not produce the same derived key across entries or across separate exports of the same application.

### 10.3 Memory Handling

Implementations should zero out decrypted payload buffers as soon as the APK files have been extracted to disk. The reference implementation fills the decrypted payload byte array with zeros immediately after extraction.

### 10.4 Plaintext Metadata in Encrypted Archives

The `header.json` entry in encrypted archives contains the package name, version, application label, and icon availability flag in plaintext. This is intentional: it allows display of application identity without a password. Implementations and users should be aware that this metadata is not protected from observation.

### 10.5 Lack of Built-in Ciphertext Integrity

Because the format utilizes AES-CBC, the encrypted blob does not include a Message Authentication Code (MAC). While an incorrect password will usually cause a padding error, it is mathematically possible for a corrupted or tampered ciphertext to produce valid padding with invalid internal plaintext. Therefore, readers must robustly handle parsing errors of the decrypted plaintext (such as malformed JSON or corrupted ZIP headers).

AES-GCM was deliberately not chosen despite providing built-in authentication, because GCM's 12-byte nonce and authentication tag alter the binary layout in a way that would make the blob format ambiguous without a version field inside the blob itself. Keeping CBC with an explicit 16-byte IV results in a simpler, fixed-offset layout that is easier to implement correctly across platforms.

---

## 11. Interoperability

An APKv file is a valid ZIP archive and can be opened by any ZIP reader. However, the encrypted entries (`manifest.enc`, `icon.enc`, `payload.enc`) are opaque binary blobs and cannot be interpreted without implementing the decryption scheme described in Section 5. Plain APKv archives expose their APK splits directly as ZIP entries named with the `.apk` extension.

### 11.1 Common Interoperability Pitfalls

The following mistakes have been observed in independent implementations and are explicitly called out to help developers avoid them.

**Using the wrong cipher mode.** The cipher is AES-256-CBC with PKCS#5 padding, not AES-GCM, AES-CTR, or any other mode. An implementation that encrypts with GCM will produce a `manifest.enc` or `payload.enc` that no other conforming reader can decrypt, even if the password is correct. The symptom is a `WRONG_PASSWORD` error on the reader side despite the password being valid.

**Partial reads of the salt and IV.** The blob header is 32 bytes: 16 bytes of salt followed by 16 bytes of IV. Both fields must be read in their entirety before key derivation begins. Using a non-blocking or single-shot read (e.g., `InputStream.read(byte[])` in Java without `DataInputStream.readFully`) may silently read fewer bytes than requested on buffered or network-backed streams. The result is a truncated salt or IV, which causes key derivation to produce a completely different key, leading to a decryption failure that is indistinguishable from a wrong password. See Section 5.2 for platform-specific guidance.

**Assuming a non-throwing decrypt means success.** On Android, `CipherInputStream` with AES-CBC may not throw `BadPaddingException` when decryption fails due to a wrong password. The stream may return corrupt bytes silently. Implementations must validate the plaintext structure (JSON validity and `format` field for `manifest.enc`; ZIP magic bytes for `payload.enc`) after decryption and must not treat a non-exception exit as proof of correct decryption.

**Sharing crypto constants between encryption and decryption paths.** If an implementation defines its crypto parameters (cipher algorithm, key length, iteration count, salt/IV sizes) in one place and both the encrypt and decrypt paths reference that same definition, a later change to the constants will automatically keep both paths consistent. If the constants are duplicated across files or classes, a change in one place without the other will silently break interoperability. A single source of truth for all crypto constants is strongly recommended.

---

## 12. Changelog

| Version | Changes                                                                                                                                                                                  |
|---------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| 1       | Initial specification. Defines plain and encrypted layouts, icon support, and PBKDF2/AES-CBC encryption scheme. Added Section 5.2 partial-read guidance, Section 5.4 CipherInputStream note, Section 10.5 GCM rationale, and Section 11.1 common pitfalls. |
