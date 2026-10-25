# A09:2025 Security logging and alerting failures

# A10:2025 Mishandling of exceptional conditions

Record privilege and money changes in an audit log that omits secrets. When a check throws, the parser fails, or a dependency is down, deny the action. The client receives a generic error. The stack stays on the server.

CWE-778 (insufficient logging), CWE-532 (secrets in logs), CWE-755 (exception handling), CWE-636 (fail open).

## Audit the change, not the secret

Write actor, action, target, and result for role changes, password resets, coupon credits, and transfers. Do not write the password, the new password, the PAN, the SSN, the session token, or the webhook secret.

```java
audit.record(session.userId(), "role.change", targetUserId, newRole);
```

```java
users.setRole(targetUserId, newRole); // no audit record
```

```python
audit.write(actor=g.user_id, action="transfer", target=recipient, cents=cents, ok=True)
```

```python
logger.info("reset %s %s", user_id, new_password)
```

What goes wrong: a role change with no record cannot be alerted on, and a password in the log line is copied into every collector. The audit call sits next to the state change, in the same transaction when the store allows it, so a rolled-back transfer does not look applied.

## Deny when parsing fails

A token, signature, or amount that does not parse is a rejection. The catch block does not attach a default user, an admin role, or a zero that later passes a check.

```typescript
let claims: Claims;
try {
  claims = authenticate(token, secret);
} catch {
  throw new Unauthorized();
}
```

```typescript
let claims: Claims = { sub: "anonymous", role: "admin" };
try {
  claims = authenticate(token, secret);
} catch {
  // keep the default
}
```

```python
try:
    claims = authenticate(token, secret)
except Exception:
    abort(401)
```

```python
try:
    claims = authenticate(token, secret)
except Exception:
    claims = {"role": "admin"}
```

What goes wrong: the exception path is the success path. A truncated token, a bad character, or a thrown clock check grants the default principal. Return 401 for a bad credential and 400 for a malformed amount. Do not include the exception text.

## Deny when the auth dependency is down

If the service that confirms a session or a role cannot be reached, the privileged handler stops. Readiness can report unhealthy. It does not return "allowed."

```java
Decision decision;
try {
    decision = authClient.decide(session.userId(), action);
} catch (AuthUnavailable ex) {
    throw new ServiceUnavailable(); // action not performed
}
if (!decision.allowed()) {
    throw new Forbidden();
}
```

```java
try {
    return authClient.decide(session.userId(), action).allowed();
} catch (AuthUnavailable ex) {
    return true;
}
```

```csharp
try {
    return await auth.AllowsAsync(userId, action);
} catch (HttpRequestException) {
    return false;
}
```

What goes wrong: an outage becomes full access. Callers should see a failure, and the state change should not run. A health check that fails open is the same bug.

## A bad divisor is a client error

Validate the divisor before you divide. On an unexpected exception, log it with the request id and respond with the generic body from configuration.md.

```python
def ratio(part: int, whole: int) -> float:
    if whole == 0:
        abort(400)
    return part / whole
```

```python
try:
    return part / whole
except Exception as exc:
    return {"stack": traceback.format_exc()}, 500
```

```java
if (whole == 0) {
    throw new BadRequest();
}
return part / whole;
```

What goes wrong: the arithmetic fault is turned into a stack trace response. The client learns internals, and the handler has no input check. Validate first, log server-side, and keep the response generic.
