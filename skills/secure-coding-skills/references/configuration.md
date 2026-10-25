# A02:2025 Security misconfiguration

Ship the production process with the extra surfaces turned off. Debug helpers, empty console passwords, and reflected CORS are configuration, and they are still defects when the production profile contains them.

CWE-209 (error content), CWE-489 (debug code), CWE-942 (permissive CORS), CWE-807 (trust of an untrusted source), CWE-798 (empty or hardcoded credential).

## Clients see a generic error

Log the exception with a request id on the server. The response body is a stable message plus that id.

```typescript
app.use((err, req, res, next) => {
  const requestId = req.id;
  logger.error({ err, requestId });
  res.status(500).json({ error: "internal", requestId });
});
```

```typescript
res.status(500).json({ error: err.message, stack: err.stack });
```

What goes wrong: the client learns internal paths, queries, and hostnames from the stack. A handler that always does this is misconfigured even when the throw is rare.

## Debug and environment routes stay out of production

Register debug dumps and raw environment routes only inside a factory branch that the production entrypoint does not call.

```python
def create_app(env: str) -> Flask:
    app = Flask(__name__)
    app.register_blueprint(api)
    if env == "local":
        app.register_blueprint(debug_routes)
    return app
```

```python
@app.route("/debug/env")
def debug_env():
    return dict(os.environ)
```

What goes wrong: the route is part of the production module, so anyone who can reach the app can read configuration. That is a misconfiguration, even if other routes require a session.

## CORS is an allowlist

Compare the request origin to a fixed list. Send `Access-Control-Allow-Credentials: true` only after that comparison succeeds. Do not copy the Origin header into the response, and do not combine credentials with `*`.

```java
if (allowedOrigins.contains(origin)) {
    response.setHeader("Access-Control-Allow-Origin", origin);
    response.setHeader("Access-Control-Allow-Credentials", "true");
    response.setHeader("Vary", "Origin");
}
```

```java
response.setHeader("Access-Control-Allow-Origin", request.getHeader("Origin"));
response.setHeader("Access-Control-Allow-Credentials", "true");
```

## Identity comes from the session

After authentication, read the user id and role from the server-side session or the verified token. Drop client headers that claim to be the user.

```csharp
var userId = session.UserId;
```

```csharp
var userId = Request.Headers["X-User-Id"].ToString();
```

What goes wrong: any client can set that header. A proxy may add identity headers, and the app should accept them only from a hop it trusts, then still map them through server-side state.

## Embedded consoles are not public and not empty

An H2 (or similar) console belongs to a local profile. Shared and production profiles leave it disabled. The database password comes from a secret store and is not blank.

```
# production
spring.h2.console.enabled=false
```

```
spring.h2.console.enabled=true
spring.datasource.username=sa
spring.datasource.password=
```

What goes wrong: the console is a database admin UI. An empty password on a reachable port is a default credential.

## Do not render secrets or internal hosts

HTML and JSON contain the fields the screen is specified to show. A team shared secret, a connection string, and an internal hostname are not fields unless the product requirement says that principal may see them.

```java
return Map.of("accountId", account.id(), "balance", money.format(account.cents()));
```

```java
return Map.of("jdbc", env.get("DB_URL"), "host", InetAddress.getLocalHost().getHostName());
```
