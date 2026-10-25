# PHP

Use the category pages for the rule. This page is the PHP API. Map bugs to the same OWASP Top 10:2025 ids: queries, HTML, shell, and paths are A05; password hashes are A04; sessions are A07; `unserialize` is A08; `display_errors` and logging secrets are A02 and A09.

## Queries bind parameters (A05)

Prepare the statement, then pass values to `execute`. Do not build the SQL string with interpolation, and do not treat `mysqli_real_escape_string` as a substitute for a bound parameter.

```php
$stmt = $pdo->prepare('SELECT id FROM ledger WHERE id = ? AND owner_id = ?');
$stmt->execute([$id, $ownerId]);
```

```php
$pdo->query("SELECT id FROM ledger WHERE id = '$id'");
```

What goes wrong: the database parses the caller's characters as SQL. PDO's default emulate-prepares setting still needs a real placeholder in the SQL text. A value concatenated before `prepare` is not bound.

## HTML is escaped (A05)

Echo text through `htmlspecialchars` with `ENT_QUOTES` and UTF-8. A template that auto-escapes is the same rule. Do not print a stored note or a query value raw.

```php
echo htmlspecialchars($note, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8');
```

```php
echo $note;
```

What goes wrong: the browser treats the note as markup. `ENT_QUOTES` covers attributes as well as element text.

## Commands are an argument list (A05)

Pass the executable and each argument separately. `escapeshellarg` around a string you then hand to `shell_exec` is still a shell. Reject a host that is not a hostname before you pass it.

```php
if (!preg_match('/\A[A-Za-z0-9.-]{1,253}\z/', $host)) {
    throw new InvalidArgumentException('host');
}
$proc = proc_open(['ping', '-c', '1', $host], $descriptors, $pipes);
```

```php
shell_exec('ping -c 1 ' . $host);
```

What goes wrong: `shell_exec`, `system`, backticks, and `exec` of a single string invoke a shell.

## Paths stay under a root (A05)

Resolve the storage root with `realpath`. Reject a name that is absolute or that resolves outside that root. Store uploads under a generated id; keep the client name as display text only.

```php
$root = realpath($storage);
$full = realpath($root . DIRECTORY_SEPARATOR . $name);
$prefix = $root . DIRECTORY_SEPARATOR;
if ($full === false || !str_starts_with($full, $prefix)) {
    throw new RuntimeException('path');
}
```

```php
readfile($storage . '/' . $name);
include $page; // $page came from the query string
```

What goes wrong: `include` and `readfile` will open whatever path the string names. A user-controlled include is code execution as well as a file read.

## Passwords and sessions (A04, A07)

Hash with `password_hash` and `PASSWORD_DEFAULT`. Verify with `password_verify`. Call `session_regenerate_id(true)` when login succeeds. Set the session cookie HttpOnly and Secure, and turn `display_errors` off in production.

```php
$hash = password_hash($password, PASSWORD_DEFAULT);
session_set_cookie_params(['httponly' => true, 'secure' => true, 'samesite' => 'Lax']);
session_regenerate_id(true);
```

```php
$hash = md5($password);
ini_set('display_errors', '1');
```

What goes wrong: MD5 is not a password hash. `display_errors` sends filesystem paths and SQL text to the client. A session id issued before login and kept after login is the caller's old id.

## Bytes become arrays, not objects (A08)

Decode JSON into arrays and map fields yourself. Do not `unserialize` a cookie, a cache entry, or a request body. Load XML with `LIBXML_NONET` so the parser does not fetch remote entities.

```php
$data = json_decode($raw, true, 16, JSON_THROW_ON_ERROR);
```

```php
$data = unserialize($raw);
```

What goes wrong: `unserialize` can construct types named by the payload. JSON gives you arrays and scalars.

## TLS and logs (A02, A09)

Leave certificate verification on. Log the account id and the result, not the password.

```php
curl_setopt($ch, CURLOPT_SSL_VERIFYPEER, true);
error_log('login_failed user=' . $userId);
```

```php
curl_setopt($ch, CURLOPT_SSL_VERIFYPEER, false);
error_log('login ' . $email . ' ' . $password);
```
