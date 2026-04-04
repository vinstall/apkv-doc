# APKv File Format Specification

Version: 2  
Status: Stable  
License: [Public Domain (Creative Commons Zero v1.0 Universal)](https://creativecommons.org/publicdomain/zero/1.0/)

---

## 1. Overview

APKv is a container format for archiving and distributing Android application packages. It encapsulates one or more APK split files together with a structured manifest, an application icon, and an optional encryption layer. The format is designed to be self-describing, transport-friendly, and protected through password-based encryption.

APKv originated in and is maintained alongside VInstall, an Android application installer. VInstall serves as the reference implementation of this specification. The format is defined independently so that any conforming tool may produce or consume APKv archives without depending on VInstall.

---

## 2. File Extension and MIME Type

| Property   | Value                       |
|------------|-----------------------------|
| Extension  | `.apkv`                     |
| MIME type  | `application/vnd.apkv`      |

---

## 3. Container Structure

An APKv file is a ZIP archive (PKWARE ZIP, ZIP64 compatible). All entries reside at the root level of the archive with no directory nesting.

All ZIP entries must use either Store (method 0) or Deflate (method 8) compression. Other compression methods are not permitted. Implementations must reject or warn on archives that contain entries using any other compression method, as they cannot be reliably decompressed across all conforming platforms.

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

| Field              | Type             | Required | Description                                                                                      |
|--------------------|------------------|----------|--------------------------------------------------------------------------------------------------|
| `format`           | string           | Yes      | Literal value `"apkv"`                                                                           |
| `formatVersion`    | integer          | Yes      | Format version number. Current value: `2`                                                        |
| `packageName`      | string           | Yes      | Android package name (e.g. `com.example.app`)                                                    |
| `versionName`      | string           | Yes      | Human-readable version string (e.g. `2.1.0`)                                                    |
| `versionCode`      | integer          | Yes      | Numeric version code as defined in the Android manifest                                          |
| `label`            | string           | Yes      | Localized application display name. Acts as the fallback when `labels` is absent or the requested locale is not present in it. |
| `labels`           | object           | No       | Multi-locale application name map. Keys must be BCP 47 language tags (e.g. `"en"`, `"id"`, `"zh-Hant"`). Values are the localized display name strings for the corresponding locale. See Section 4.1.1 for locale resolution behavior. |
| `isSplit`          | boolean          | Yes      | `true` if the application uses split APK delivery                                                |
| `splits`           | array            | Yes      | Array of APK filenames present in the archive                                                    |
| `encrypted`        | boolean          | Yes      | `true` if the archive payload is encrypted                                                       |
| `hasIcon`          | boolean          | Yes      | `true` if an icon entry is present in the archive                                                |
| `exportedAt`       | integer (64-bit) | Yes      | Unix epoch timestamp in milliseconds at export time                                              |
| `minSdkVersion`    | integer          | Yes      | Minimum Android SDK version required to install the application (e.g. `21` for Android 5.0)     |
| `targetSdkVersion` | integer          | Yes      | Android SDK version the application was compiled and tested against (e.g. `34` for Android 14)  |
| `checksums`        | object           | No       | SHA-256 hash map keyed by APK filename. Values must follow the form `"sha256:<hex>"` where `<hex>` is the lowercase 64-character hexadecimal digest of the raw APK entry bytes. See Section 4.1.2 for verification behavior. |
| `totalSize`        | integer (64-bit) | No       | Sum of the decompressed sizes of all APK entries in bytes. Represents uncompressed storage requirements, not the size of the archive on disk. Allows installer UIs to display storage requirements before extraction. |
| `permissions`      | array of strings | No       | Android permission strings declared in the base APK's manifest (e.g. `"android.permission.INTERNET"`). Allows host applications to display a permission summary before installation without unpacking the APK. This list reflects declarations only and does not imply that all listed permissions will be granted at runtime. |
| `_comment`         | string           | No       | Human-readable note for tooling authors or archive inspectors. See Section 4.5.1 for the `_comment` convention. Readers must silently ignore this field. |

Example:

```json
{
  "_comment": "Exported by ExampleTool 3.0 on 2025-04-03",
  "format": "apkv",
  "formatVersion": 2,
  "packageName": "com.example.app",
  "versionName": "2.1.0",
  "versionCode": 210,
  "label": "Example App",
  "labels": {
    "en": "Example App",
    "id": "Aplikasi Contoh",
    "zh-Hant": "範例應用程式"
  },
  "isSplit": true,
  "splits": ["base.apk", "split_config.arm64_v8a.apk"],
  "encrypted": false,
  "hasIcon": true,
  "exportedAt": 1743667200000,
  "minSdkVersion": 21,
  "targetSdkVersion": 34,
  "checksums": {
    "base.apk": "sha256:e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855",
    "split_config.arm64_v8a.apk": "sha256:a665a45920422f9d417e4867efdc4fb8a04a1f3fff1fa07e998e86f7f7a27ae3"
  },
  "totalSize": 67108864,
  "permissions": [
    "android.permission.INTERNET",
    "android.permission.RECEIVE_BOOT_COMPLETED"
  ]
}
```

#### 4.1.1 Locale Resolution for `labels`

When resolving a display name for a given locale, readers must apply the following fallback chain in order:

1. Look up the exact BCP 47 locale tag in `labels` (e.g. `"zh-Hant"`).
2. If not found, look up the base language subtag (e.g. `"zh"`).
3. If still not found, or if `labels` is absent, use the `label` field.

The `label` field is always required and must contain a meaningful default display name. Implementations must not leave `label` empty and rely solely on `labels`.

Writers must use well-formed BCP 47 tags as defined in [RFC 5646](https://www.rfc-editor.org/rfc/rfc5646). Language tags are case-insensitive for lookup purposes; readers should normalize tags to lowercase before performing map lookups to avoid case-mismatch failures across writers that differ in capitalization convention (e.g. `"zh-hant"` vs `"zh-Hant"`).

#### 4.1.2 Checksum Verification

The `checksums` object is optional. When present, each key must be the filename of an APK entry listed in `splits`, and each value must be a string of the form `"sha256:<hex>"` where `<hex>` is exactly 64 lowercase hexadecimal characters representing the SHA-256 digest of the raw, unmodified APK entry bytes as stored in the archive (before any ZIP decompression if the entry is stored with deflate compression — the digest is computed over the decompressed bytes).

**Reader behavior:**

- Verification is optional but strongly recommended. Readers that skip verification must not silently suppress the fact that checksums were present.
- If a reader performs verification and the computed digest of an extracted APK does not match the declared value, it must surface a visible warning to the user identifying the specific filename and the mismatch. It must not silently ignore the discrepancy or continue installation without user acknowledgment.
- A mismatch is not grounds for automatic silent discard. The user must be informed and given the opportunity to decide how to proceed.
- If an entry is listed in `splits` but has no corresponding key in `checksums`, verification for that entry is simply skipped. A partial `checksums` object is valid.
- If a key appears in `checksums` that does not correspond to any entry in `splits`, readers must ignore that key without error.

Writers must compute the digest over the fully decompressed APK bytes, not over the compressed ZIP entry bytes.

### 4.2 header.json

Present only in encrypted archives. A UTF-8 encoded JSON object that provides minimal metadata readable without a password. This allows a host application to display the application name, package, and version while prompting the user for a password.

| Field           | Type             | Required | Description                                                                                      |
|-----------------|------------------|----------|--------------------------------------------------------------------------------------------------|
| `packageName`   | string           | Yes      | Android package name                                                                             |
| `versionName`   | string           | Yes      | Human-readable version string                                                                    |
| `label`         | string           | Yes      | Localized application display name. Fallback when `labels` is absent or the locale is not found. |
| `labels`        | object           | No       | Multi-locale application name map. Same structure and BCP 47 key convention as `manifest.json`. Locale resolution follows Section 4.1.1. Writers should mirror the `labels` value from `manifest.json` verbatim so pre-password display reflects the same locale data as the decrypted manifest. |
| `encrypted`     | boolean          | Yes      | Always `true` in this context                                                                    |
| `hasIcon`       | boolean          | Yes      | `true` if `icon.enc` is present                                                                  |
| `exportedAt`    | integer (64-bit) | No       | Unix epoch timestamp in milliseconds at export time. When present, must be identical to the `exportedAt` value in `manifest.json`. Exposed here so host applications can display export time prior to decryption. |
| `_comment`      | string           | No       | Human-readable note for tooling authors or archive inspectors. See Section 4.5.1 for the `_comment` convention. Readers must silently ignore this field. |

### 4.3 icon.webp

A WebP-encoded image representing the application launcher icon. The image must be square and is recommended to be encoded at 192x192 pixels using lossy compression at quality 80. This size provides sufficient visual fidelity for display purposes while keeping file size minimal.

Implementations may choose not to include this entry. The `hasIcon` field in `manifest.json` or `header.json` indicates whether the entry is present.

### 4.4 APK Split Files (plain archives)

Each `.apk` file entry corresponds to a standard Android APK file. Filenames must match the entries listed in the `splits` array of `manifest.json`. The base APK is typically named `base.apk` or uses the original filename from the source installation.

### 4.5 Unknown Fields

Readers must silently ignore any unrecognized fields encountered in `manifest.json` or `header.json`. Implementations must not treat the presence of an unknown field as a parse error or as a signal of format incompatibility. This rule ensures forward compatibility when new optional fields are introduced in future amendments without a `formatVersion` increment.

#### 4.5.1 The `_comment` Convention

Since JSON does not natively support comments, writers may include a `_comment` string field at the top level of `manifest.json` and `header.json` to embed human-readable notes. Typical uses include recording the name and version of the tool that produced the archive, the export date, or other diagnostically useful information.

Readers must silently ignore the `_comment` field under the same rule as any other unknown field. Writers must not place structured or machine-readable data inside `_comment`; it is intended solely for human inspection.

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

**Writer note — streaming vs. in-memory encryption:** Writers may produce the ciphertext using either an in-memory `doFinal` call or a streaming `update`/`doFinal` loop. Both approaches produce an identical binary blob provided the same cipher algorithm, key, and IV are used. Streaming encryption is explicitly permitted and is the recommended approach for `payload.enc`, which may contain large APK files that are impractical to hold entirely in memory. `manifest.enc` and `icon.enc` are small and may be encrypted in-memory for simplicity. Regardless of the approach chosen by the writer, the resulting blob format is always `[ salt ][ iv ][ ciphertext ]` and is consumed identically by any conforming reader.

**Critical implementation note — reading salt and IV:** When *reading* (decrypting) the salt and IV from a byte stream, implementations must use a blocking read that guarantees all requested bytes are consumed before proceeding. On many platforms and I/O layers (particularly Android's `InputStream` and Python's file objects in buffered mode), a single `read(n)` call is not guaranteed to return exactly `n` bytes even when more data is available. If a partial read occurs, the derived key and IV will be wrong and decryption will silently fail or produce corrupt output. Use the appropriate primitive for your platform:

| Platform / Language | Correct primitive                                                                                  |
|---------------------|----------------------------------------------------------------------------------------------------|
| Java / Kotlin       | `DataInputStream.readFully(buf)`                                                                   |
| Python              | `file.read(n)` on a real file object is safe; for sockets or pipes use `socket.recv_exactly` or loop until full |
| C / C++             | Loop `read()` until all bytes received                                                             |
| Go                  | `io.ReadFull(r, buf)`                                                                              |
| Rust                | `Read::read_exact(&mut buf)`                                                                       |

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

The `formatVersion` field is a monotonically increasing integer. Version `1` is the baseline; version `2` is the current version defined by this document. Future versions may introduce additional entries or manifest fields. Readers encountering an unknown `formatVersion` value should treat the file as potentially incompatible and inform the user accordingly, but should attempt to parse known fields and surface a compatibility warning to the user.

---

## 10. Security Considerations

### 10.1 Password Strength

The KDF iteration count of 120,000 is chosen to impose a meaningful computational cost on brute-force attacks while remaining acceptable on mobile hardware. Users should be advised that the security of an encrypted APKv archive is bounded by the entropy of the chosen password.

### 10.2 No Key Derivation Reuse

Each encrypted entry uses a freshly generated random salt, ensuring that identical passwords do not produce the same derived key across entries or across separate exports of the same application.

### 10.3 Memory Handling

Implementations should zero out decrypted payload buffers as soon as the APK files have been extracted to disk. VInstall (the reference implementation) fills the decrypted payload byte array with zeros immediately after extraction.

### 10.4 Plaintext Metadata in Encrypted Archives

The `header.json` entry in encrypted archives contains the package name, version, application label, locale map, export timestamp, and icon availability flag in plaintext. This is intentional: it allows display of application identity without a password. Implementations and users should be aware that this metadata is not protected from observation. In particular, the addition of `labels` in v2 means that locale-specific display names for all included locales are also exposed in plaintext; writers should consider this when deciding how much locale coverage to include in `header.json`.

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

**Sharing crypto constants between encryption and decryption paths.** If an implementation defines its crypto parameters (cipher algorithm, key length, iteration count, salt/IV sizes) in one place and both the encrypt and decrypt paths reference that same definition, a later change to the constants will automatically keep both paths consistent. If the constants are duplicated across files or classes, a change in one place without the other will silently break interoperability. A single source of truth for all crypto constants is strongly recommended. Note that this pitfall concerns *constants* (algorithm identifiers, sizes, iteration counts), not *implementation strategy*: using a streaming `cipher.update` loop for `payload.enc` encryption while using an in-memory `cipher.doFinal` call for `manifest.enc` is explicitly valid, provided both paths reference the same constants and produce the same `[ salt ][ iv ][ ciphertext ]` blob layout.

**Computing checksums over compressed bytes.** When populating the `checksums` field, writers must hash the decompressed APK bytes, not the raw compressed entry bytes as stored inside the ZIP. A reader extracting the APK will naturally produce the decompressed bytes; if the writer hashed compressed data instead, the digest will never match on any conforming reader regardless of data integrity.

**Case-sensitive locale lookup in `labels`.** BCP 47 tags are case-insensitive by convention, but JSON object keys are case-sensitive. Implementations that perform exact-match key lookup without first normalizing both the query locale and the map keys to lowercase will silently fall back to `label` for locales that differ only in case (e.g. `"zh-Hant"` vs `"zh-hant"`). Normalize to lowercase before lookup.

---

## 12. Changelog

| Version | Changes |
|---------|---------|
| 2 | Bumped `formatVersion` to `2`. Added `minSdkVersion` (required integer) and `targetSdkVersion` (required integer) to `manifest.json`. Added `labels` (optional BCP 47 locale-to-name map) to both `manifest.json` and `header.json` with specified fallback behavior to `label`. Added `checksums` (optional SHA-256 hash map keyed by APK filename) to `manifest.json` with specified verification and mismatch-warning behavior. Added `exportedAt` (optional 64-bit Unix millisecond timestamp) to `header.json`. Added optional `totalSize` field (sum of decompressed APK entry sizes in bytes) and optional `permissions` field (declared Android permission strings; does not imply runtime grant) to `manifest.json`. Added `_comment` string field convention to `manifest.json` and `header.json` for human-readable notes. Added Section 3 requirement that all ZIP entries must use Store (method 0) or Deflate (method 8) compression only. Added Section 4.5 (Unknown Fields) specifying that readers must silently ignore unrecognized fields for forward compatibility. Updated Section 9 versioning language. Expanded Section 10.4 to note that `labels` and `exportedAt` are exposed in plaintext via `header.json`. Added two entries to Section 11.1 covering checksum hashing of compressed vs. decompressed bytes and case-sensitive locale lookup. Clarified Section 1 to identify VInstall as the origin and reference implementation of the format. Clarified Section 5.2 to explicitly permit streaming encryption for `payload.enc` and to scope the blocking-read requirement to the decryption (reader) side only. Updated Section 10.3 to name VInstall explicitly instead of referring to it anonymously as "the reference implementation". Clarified the §11.1 crypto-constants pitfall to distinguish between constant reuse (required) and implementation strategy (streaming vs. in-memory, both valid). |
| 1 | Initial specification. Defines plain and encrypted layouts, icon support, and PBKDF2/AES-CBC encryption scheme. Added Section 5.2 partial-read guidance, Section 5.4 CipherInputStream note, Section 10.5 GCM rationale, and Section 11.1 common pitfalls. |