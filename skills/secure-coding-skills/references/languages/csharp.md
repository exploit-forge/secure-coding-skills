# C#

.NET already appears in the category pages (`SqlCommand` parameters, `Html.Raw`, `ProcessStartInfo`, `BinaryFormatter`, cookie flags). This page is the set to open for a C# review so those APIs are not left as one-line mentions.

## Authorize the resource (A01)

Put `[Authorize]` on the controller or the endpoint, and add a role policy for admin routes. Load the row with the caller's id. Do not trust `X-User-Id`. Over-posting is the same bug: bind a DTO that has only the fields the caller may set, not the entity that includes `Role` or `Balance`.

```csharp
[Authorize]
[HttpGet("{id}")]
public async Task<ActionResult<Account>> Get(string id)
{
    var account = await db.Accounts
        .SingleOrDefaultAsync(a => a.Id == id && a.OwnerId == User.GetUserId());
    return account is null ? NotFound() : Ok(account);
}
```

```csharp
[AllowAnonymous]
[HttpGet("{id}")]
public async Task<Account> Get(string id) => await db.Accounts.FindAsync(id);
```

```csharp
public record ProfileUpdate(string DisplayName, string Email);
await db.Users.Where(u => u.Id == userId)
    .ExecuteUpdateAsync(s => s.SetProperty(u => u.DisplayName, body.DisplayName));
```

```csharp
db.Entry(user).CurrentValues.SetValues(Request.Body); // role and balance included
```

What goes wrong: `FindAsync(id)` returns any row to any caller who can hit the route. Binding the entity from the body writes columns the form never showed.

## SQL parameters and EF (A05)

Use parameters for ADO.NET. Use LINQ or `FromSqlInterpolated` for EF. `FromSqlRaw` with concatenation is still SQL injection. Column names for `ORDER BY` come from an allowlist, not from the query string.

```csharp
cmd.CommandText = "SELECT id FROM ledger WHERE id = @id AND owner_id = @owner";
cmd.Parameters.AddWithValue("@id", id);
cmd.Parameters.AddWithValue("@owner", user.Id);
```

```csharp
var rows = db.Ledger.FromSqlRaw(
    $"SELECT * FROM ledger WHERE id = '{id}'");
```

What goes wrong: an interpolated raw query is sent as SQL text. `FromSqlInterpolated` parameterizes the holes in the interpolated string; concatenated `FromSqlRaw` does not.

## Razor encodes (A05)

`@Model.Note` encodes. `Html.Raw` and `MarkupString` in Blazor do not. Use them only for markup you produced.

```html
<p>@Model.Note</p>
```

```html
<p>@Html.Raw(Model.Note)</p>
```

What goes wrong: stored notes render as HTML for the next user.

## Processes and paths (A05)

Set `UseShellExecute` to false and add each argument to `ArgumentList`. For files, combine under a rooted full path and reject when the result is not under that root. Store uploads under `Guid.NewGuid()`, not `FileName`.

```csharp
var start = new ProcessStartInfo { FileName = "ping", UseShellExecute = false };
start.ArgumentList.Add("-n");
start.ArgumentList.Add("1");
start.ArgumentList.Add(host);
Process.Start(start);
```

```csharp
Process.Start(new ProcessStartInfo("ping", "-n 1 " + host) { UseShellExecute = true });
```

```csharp
var root = Path.GetFullPath(storage);
var full = Path.GetFullPath(Path.Combine(root, name));
if (!full.StartsWith(root + Path.DirectorySeparatorChar, StringComparison.Ordinal))
    throw new InvalidOperationException("path");
```

```csharp
File.ReadAllBytes(Path.Combine(storage, name));
```

What goes wrong: `UseShellExecute` true hands the string to the shell. `Path.Combine` ignores earlier parts when a later part is rooted, so a caller path can leave the upload directory.

## Passwords, data protection, TLS (A04, A07)

Use `PasswordHasher<T>` or an Argon2 package. Do not store MD5 or SHA-256 of the password. Session and auth cookies set `HttpOnly` and `Secure`. Do not return `true` from `ServerCertificateCustomValidationCallback`.

```csharp
var hash = hasher.HashPassword(user, password);
options.Cookie.HttpOnly = true;
options.Cookie.SecurePolicy = CookieSecurePolicy.Always;
```

```csharp
var hash = Convert.ToHexString(MD5.HashData(Encoding.UTF8.GetBytes(password)));
handler.ServerCertificateCustomValidationCallback = (_, _, _, _) => true;
```

What goes wrong: MD5 is fast to guess offline. A callback that always returns true accepts any certificate.

## Deserialization (A08)

Deserialize JSON into a DTO. Do not use `BinaryFormatter`. Leave Newtonsoft `TypeNameHandling` at `None`. Do not point `SoapFormatter` or `LosFormatter` at a request body.

```csharp
var payment = JsonSerializer.Deserialize<PaymentDto>(json);
```

```csharp
var graph = new BinaryFormatter().Deserialize(stream);
JsonConvert.DeserializeObject(json, new JsonSerializerSettings {
    TypeNameHandling = TypeNameHandling.All
});
```

What goes wrong: the payload chooses the type. A DTO does not. Upgrade `Newtonsoft.Json` off 12.0.3 as the dependency page says, and keep type names off even after the upgrade.

## Secrets and failures (A02, A09, A10)

Read connection strings from configuration or user-secrets / Key Vault, not from a committed `appsettings.json`. Log the user id, not the password. If the auth service throws, deny the action. Do not return `ex.ToString()` from an API.

```csharp
logger.LogInformation("Login failed for {UserId}", userId);
return Problem(statusCode: StatusCodes.Status500InternalServerError, title: "internal");
```

```csharp
logger.LogInformation("Login {Email} {Password}", email, password);
return StatusCode(500, ex.ToString());
```

What goes wrong: the response and the log become a copy of the secret and of the stack. A catch that returns success when the identity provider is down grants access during an outage.
