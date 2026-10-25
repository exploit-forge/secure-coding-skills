# A04:2025 Cryptographic failures

Store a password hash from a password hashing function. Encrypt with an authenticated mode and a key from a secret store. API responses that hold ciphertext do not also hold the full PAN or SSN.

CWE-916 (password hash), CWE-327 (broken mode), CWE-798 (hardcoded key), CWE-312 (cleartext sensitive data).

## Password hashes

Use Argon2id or bcrypt through the platform library. MD5 and a single SHA-256 of the password are not password hashes. On login, verify with the password hasher. If you must read an old MD5 row, verify it once, then replace the stored value with the real hash in the same request, and delete the MD5 path when no rows remain.

```java
String hash = passwordEncoder.encode(password); // bcrypt or argon2
```

```java
MessageDigest.getInstance("MD5").digest(password.getBytes(StandardCharsets.UTF_8));
```

```python
from argon2 import PasswordHasher
ph = PasswordHasher()
stored = ph.hash(password)
```

```csharp
var hash = hasher.Hash(password); // ASP.NET Identity PasswordHasher or an Argon2 package
```

What goes wrong: MD5 is fast and, in the form we keep seeing, unsalted. A copied user table can be guessed offline. The login path that still calls MD5 keeps that exposure alive.

## Authenticated encryption, key outside the source tree

Use AES-GCM (or the platform's authenticated encryption API). Generate a fresh nonce per message and store it with the ciphertext. Load the key from a secret store. Do not use ECB. Do not put the key in a source file.

```java
Cipher cipher = Cipher.getInstance("AES/GCM/NoPadding");
cipher.init(Cipher.ENCRYPT_MODE, keyFromStore, new GCMParameterSpec(128, nonce));
```

```java
Cipher cipher = Cipher.getInstance("AES/ECB/PKCS5Padding");
cipher.init(Cipher.ENCRYPT_MODE, new SecretKeySpec(keyBytesFromSource, "AES"));
```

```csharp
using var aes = new AesGcm(keyFromStore, tagSizeInBytes: 16);
aes.Encrypt(nonce, plaintext, ciphertext, tag);
```

```typescript
crypto.createCipheriv("aes-256-gcm", keyFromStore, nonce);
```

What goes wrong: ECB repeats blocks, so equal chunks of a PAN stay equal in ciphertext. A key in the repository is available to everyone who can read the repo, including other environments that copied the same literal.

## Do not return PAN or SSN with the ciphertext

Persist ciphertext. If the UI needs a hint, persist and return a separate last-four or masked value that was stored for display. Do not put the full number in the response, and do not decrypt it into JSON to build that hint on the way out.

```typescript
return { id: card.id, last4: card.last4 };
```

```typescript
return { pan: card.pan, ciphertext: card.ciphertext };
```

```java
record CardView(String id, String last4) {}
```

What goes wrong: the encryption never protects the number if the API returns the number beside the ciphertext. Logs and browser traces then hold the clear value.
