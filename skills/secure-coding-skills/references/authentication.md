# A07:2025 Authentication failures

Accept a token only after a full signature check. A missing signature, a short prefix, or a hash of the secret glued to the message is a reject. Session cookies are unreadable to script. Login failures are limited, and the password never reaches a log line. A parser error denies the request (see also logging-and-failures.md).

CWE-347 (improper signature verification), CWE-1004 (sensitive cookie), CWE-307 (excessive attempts), CWE-532 (sensitive data in logs).

## Require a real signature

A compact token has three segments. Fewer than three, including a string with no dot, is rejected before any privilege is attached. Verify HMAC (or the scheme's asymmetric signature) over the header and payload. Compare the full MAC with a constant-time compare.

```typescript
function authenticate(token: string, secret: Buffer): Claims {
  const parts = token.split(".");
  if (parts.length !== 3 || parts.some((p) => p.length === 0)) {
    throw new Unauthorized();
  }
  const mac = crypto.createHmac("sha256", secret).update(`${parts[0]}.${parts[1]}`).digest();
  const presented = decode(parts[2]);
  if (presented.length !== mac.length || !crypto.timingSafeEqual(presented, mac)) {
    throw new Unauthorized();
  }
  return readClaims(parts[1]);
}
```

```typescript
function authenticate(token: string): Claims {
  if (!token.includes(".")) {
    return JSON.parse(token) as Claims; // no signature segment, still accepted
  }
  return verifyPrefix(token);
}
```

```csharp
if (!CryptographicOperations.FixedTimeEquals(presented, fullMac)) {
    throw new UnauthorizedAccessException();
}
```

```csharp
if (presentedHex.StartsWith(fullMacHex.Substring(0, 8), StringComparison.Ordinal)) {
    return claims; // eight characters are not a signature
}
```

```python
mac = hmac.new(secret, message, hashlib.sha256).digest()
if not hmac.compare_digest(mac, presented):
    raise PermissionError("token")
```

```python
digest = hashlib.sha256(secret + body).hexdigest()  # concatenation is not an HMAC
if digest == presented:
    return claims
```

What goes wrong:

- A token with no dot never enters the verifier, and the code treats the remainder as already trusted.
- Comparing eight characters accepts any MAC that shares that prefix.
- `SHA-256(secret + body)` is not HMAC. The check can be satisfied without the construction HMAC uses, and a plain equality is not constant-time.

Check expiry and audience in the same function, after the MAC matches. Do not read claims from an unverified segment to decide whether to skip verification.

## Session cookies

Set HttpOnly so page script cannot read the session. Set Secure when the site is HTTPS. Set SameSite to Lax or Strict unless a documented cross-site flow needs otherwise.

```typescript
res.cookie("sid", sessionId, { httpOnly: true, secure: true, sameSite: "lax" });
```

```typescript
res.cookie("sid", sessionId);
```

```java
cookie.setHttpOnly(true);
cookie.setSecure(true);
```

```csharp
options.Cookie.HttpOnly = true;
options.Cookie.SecurePolicy = CookieSecurePolicy.Always;
```

What goes wrong: a cookie the browser will hand to script is readable by any markup that runs on the origin. The flag is the fix; do not also store the session id in `localStorage`.

## Lockout, and no password in the log

Count failures per account and per source, and slow or lock after a small limit. Log the account id and the result. Never log the password, the reset token, or the session id.

```python
def login(email: str, password: str):
    if locked(email):
        abort(429)
    user = users.by_email(email)
    if user is None or not hasher.verify(user.hash, password):
        register_failure(email)
        current_app.logger.info("login_failed", extra={"email": email})
        abort(401)
    clear_failures(email)
```

```python
current_app.logger.info("login %s %s", email, password)
```

```java
log.info("login failed userId={}", userId);
```

```java
log.info("login {} {}", email, password);
```

What goes wrong: unlimited attempts make guessing practical, and a password written to logs is stored in every system that collects those logs. The lockout counter itself must not record the password.
