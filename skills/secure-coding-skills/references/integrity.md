# A08:2025 Software or data integrity failures

Untrusted bytes become objects only through a parser aimed at a fixed schema. YAML safe-load, a typed JSON DTO, and a checked webhook signature are the patterns. Java and .NET object streams, and a webhook handler that never reads the signature, are the bugs.

CWE-502 (deserialization of untrusted data), CWE-345 (insufficient verification of data authenticity).

## YAML from outside the process uses a safe loader

Python uses `safe_load`. Java uses SnakeYAML 2.x (see the deny list) and does not turn global tags on for documents a caller, a partner, or a repo file can change.

```python
data = yaml.safe_load(text)
if not isinstance(data, dict):
    raise ValueError("yaml")
```

```python
data = yaml.load(text, Loader=yaml.Loader)
```

```java
Yaml yaml = new Yaml(new SafeConstructor(new LoaderOptions()));
Object data = yaml.load(text);
```

```java
Object data = new Yaml().load(text); // 1.x constructor accepts tags the document chooses
```

What goes wrong: the document picks the type that gets constructed. A safe loader only builds basic structures (maps, lists, strings, numbers). That is what application config from an untrusted source is allowed to be. Map those structures into a DTO in your code.

## Do not revive platform objects from caller bytes

`BinaryFormatter`, `ObjectInputStream`, and `TypeNameHandling` other than `None` instantiate types named by the payload. Read JSON into a type you wrote.

```csharp
var payment = JsonSerializer.Deserialize<PaymentDto>(json);
```

```csharp
var graph = new BinaryFormatter().Deserialize(stream);
```

```java
PaymentDto payment = objectMapper.readValue(bytes, PaymentDto.class);
```

```java
Object value = new ObjectInputStream(stream).readObject();
```

```csharp
// Newtonsoft.Json: leave type names off
var payment = JsonConvert.DeserializeObject<PaymentDto>(json);
```

```csharp
JsonConvert.DeserializeObject(json, new JsonSerializerSettings {
    TypeNameHandling = TypeNameHandling.All
});
```

What goes wrong: the bytes choose a class on the classpath. The fix is a closed DTO, not a filter you hope lists every dangerous type. `ObjectInputStream` on a trusted backup you produced is still a design to retire; do not point it at uploads or message bodies.

## Webhooks check the signature, then the body

Read the raw body once. Compute HMAC with the provider secret and compare it to the signature header (`X-Hub-Signature-256` for GitHub-style hooks, or the header that provider documents) using a constant-time compare. A missing header is a reject. Parse JSON only after the compare succeeds.

```python
def webhook():
    raw = request.get_data()
    presented = request.headers.get("X-Hub-Signature-256", "")
    mac = hmac.new(secret, raw, hashlib.sha256).hexdigest()
    expected = "sha256=" + mac
    if not hmac.compare_digest(expected, presented):
        abort(401)
    handle(json.loads(raw))
```

```python
def webhook():
    handle(request.get_json())  # signature header is never read
```

```typescript
const mac = crypto.createHmac("sha256", secret).update(rawBody).digest("hex");
const expected = Buffer.from(`sha256=${mac}`);
const presented = Buffer.from(header);
if (expected.length !== presented.length || !crypto.timingSafeEqual(expected, presented)) {
  throw new Unauthorized();
}
```

What goes wrong: the handler trusts the JSON because it arrived at the webhook URL. Anyone who can reach that URL can submit a body. Verification uses the raw bytes the provider signed; re-serializing the parsed JSON will not match.

Still works: a delivery whose signature matches is handled. Add a test with a known body and a known test secret that expects HTTP 200, and a test with a missing header that expects a rejection.
