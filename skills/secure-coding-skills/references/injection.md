# A05:2025 Injection

Keep untrusted text in a parameter, an escaped text node, or a single argv element. Build SQL, HTML, shell commands, and filesystem paths with the API that treats data as data.

CWE-89, CWE-79, CWE-78, CWE-22, CWE-434.

## SQL uses the driver's placeholders

Bind every value. Interpolating a string, including an id, is the bug. Column names and `ORDER BY` fields come from an allowlist, not from the client.

```java
PreparedStatement ps = conn.prepareStatement(
    "SELECT id, amount_cents FROM ledger WHERE id = ? AND owner_id = ?");
ps.setString(1, id);
ps.setString(2, session.userId());
```

```java
conn.createStatement().executeQuery(
    "SELECT id, amount_cents FROM ledger WHERE id = '" + id + "'");
```

```csharp
cmd.CommandText = "SELECT id FROM ledger WHERE id = @id AND owner_id = @owner";
cmd.Parameters.AddWithValue("@id", id);
cmd.Parameters.AddWithValue("@owner", user.Id);
```

```python
# sqlite3 uses '?'. psycopg uses '%s'. Use the marker your driver documents.
cur.execute(
    "SELECT id FROM ledger WHERE id = ? AND owner_id = ?",
    (ledger_id, user_id),
)
```

```typescript
await pool.query(
  "SELECT id FROM ledger WHERE id = $1 AND owner_id = $2",
  [id, userId],
);
```

What goes wrong: the database parses the caller's characters as SQL. Each driver looks safe only when its own placeholder is used. A `?` sent to a server that expects `$1` is not a bound parameter.

## HTML escapes user text

Default template encoding is the pattern. Raw HTML sinks are for markup you generated, not for stored notes, names, or query strings.

```html
<!-- Java / Thymeleaf -->
<p th:text="${note}"></p>
```

```html
<p th:utext="${note}"></p>
```

```html
<!-- C# / Razor -->
<p>@Model.Note</p>
```

```html
<p>@Html.Raw(Model.Note)</p>
```

```python
# Jinja2 with autoescape on
{{ note }}
```

```python
{{ note | safe }}
```

```tsx
<p>{note}</p>
```

```tsx
<div dangerouslySetInnerHTML={{ __html: note }} />
```

Reflected and stored are the same rule. If the value is written into HTML, an attribute, or a URL from the request or from the database, it goes through the escape path. `th:utext` and `Html.Raw` skip that path, so a note saved earlier runs as markup for the next reader.

## Commands are an argument list

Start the process with a fixed executable and a list of arguments. Check that a host matches a hostname or address pattern before you pass it. Do not join it into a string that a shell parses.

```java
if (!host.matches("[A-Za-z0-9.-]{1,253}")) {
    throw new BadRequest();
}
new ProcessBuilder("ping", "-c", "1", host).start();
```

```java
Runtime.getRuntime().exec("ping -c 1 " + host);
```

```python
subprocess.run(["ping", "-c", "1", host], check=False)
```

```python
os.system("ping -c 1 " + host)
```

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

```typescript
execFile("ping", ["-c", "1", host]);
```

```typescript
exec("ping -c 1 " + host);
```

What goes wrong: `exec`, `os.system`, and `UseShellExecute` hand the string to a shell. The host is then more than an argument. `ProcessStartInfo` is only safe when `UseShellExecute` is false and the host is one argument.

## Paths stay under a root

Resolve the storage root to a real directory. Normalize the requested child and reject it when it is not under that root. Do this before you read or write.

```java
Path root = storage.toRealPath();
Path child = root.resolve(requested).normalize();
if (!child.startsWith(root)) {
    throw new Forbidden();
}
```

```java
Path child = storage.resolve(requested); // no root check
```

```python
root = Path(storage).resolve()
child = (root / requested).resolve()
if not child.is_relative_to(root):
    raise PermissionError("path")
```

```typescript
const root = path.resolve(storage);
const child = path.resolve(root, requested);
if (!child.startsWith(root + path.sep)) {
  throw new Error("path");
}
```

What goes wrong: a relative segment in the client path leaves the directory you meant to expose. Canonicalizing without the prefix check does not help.

## Uploads ignore the client filename

Store the bytes under a generated id. If you need the original name, save it as a display string in the database, not as a path segment.

```java
String stored = UUID.randomUUID().toString();
Files.copy(upload.getInputStream(), root.resolve(stored));
meta.setDisplayName(Paths.get(upload.getSubmittedFileName()).getFileName().toString());
```

```java
Files.copy(upload.getInputStream(), root.resolve(upload.getSubmittedFileName()));
```

```csharp
var stored = Guid.NewGuid().ToString("n");
await using var output = File.Create(Path.Combine(root, stored));
```

```python
stored = uuid.uuid4().hex
path = root / stored
```

What goes wrong: the submitted name can point outside the upload directory or overwrite an existing file. The display name can still be shown after you take only the final segment and treat it as text.
