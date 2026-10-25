# Go

Go's standard library is safe when you use the APIs that keep data and code apart. The categories match the rest of this skill.

## SQL uses placeholders (A05)

Pass arguments to `Query`, `QueryRow`, or `Exec`. The placeholder is `?` for `database/sql` drivers that use it, and `$1` for pgx. Do not `fmt.Sprintf` a value into the statement. Identifiers come from an allowlist.

```go
row := db.QueryRowContext(ctx,
    "SELECT id FROM ledger WHERE id = $1 AND owner_id = $2", id, ownerID)
```

```go
q := fmt.Sprintf("SELECT id FROM ledger WHERE id = '%s'", id)
row := db.QueryRowContext(ctx, q)
```

What goes wrong: the formatted string is SQL. Bound arguments are data even when they contain quotes.

## HTML uses `html/template` (A05)

Render pages with `html/template`. Reach for `text/template` only for text that is not HTML. Do not mark a user string as `template.HTML`.

```go
tpl := template.Must(template.ParseFS(files, "note.html"))
tpl.Execute(w, note) // package html/template
```

```go
tpl := texttemplate.Must(texttemplate.New("n").Parse("<p>{{.}}</p>"))
tpl.Execute(w, note)
```

What goes wrong: `text/template` copies bytes through. `template.HTML`, `template.JS`, and `template.URL` turn off escaping for that value.

## Processes take separate arguments (A05)

`exec.Command` does not use a shell. Do not pass `sh` and `-c` with a concatenated string. Check the host before you add it as an argument.

```go
if !hostname.MatchString(host) {
    return fmt.Errorf("host")
}
cmd := exec.CommandContext(ctx, "ping", "-c", "1", host)
```

```go
cmd := exec.CommandContext(ctx, "sh", "-c", "ping -c 1 "+host)
```

What goes wrong: `-c` makes the shell parse `host`. `exec.Command("ping", "-c", "1", host)` does not.

## Join, then prove the path is inside the root (A05)

`filepath.Join` and `filepath.Clean` do not enforce a root. An absolute `name` makes `Join` drop the earlier elements. Reject absolute names, clean both sides, and reject a relative path whose first element is `..`.

```go
func openUnder(root, name string) (*os.File, error) {
    if filepath.IsAbs(name) {
        return nil, fmt.Errorf("path")
    }
    root = filepath.Clean(root)
    full := filepath.Clean(filepath.Join(root, name))
    rel, err := filepath.Rel(root, full)
    if err != nil || rel == ".." || strings.HasPrefix(rel, ".."+string(os.PathSeparator)) {
        return nil, fmt.Errorf("path")
    }
    return os.Open(full)
}
```

```go
f, err := os.Open(filepath.Join(root, name))
```

What goes wrong: the joined path can sit outside `root`, and `Open` will follow it.

## Tokens and TLS (A04, A07)

Draw session ids and nonces from `crypto/rand`. Leave `InsecureSkipVerify` unset. Compare MACs with `subtle.ConstantTimeCompare` on equal lengths. Password hashes come from bcrypt or argon2, not `md5` or plain SHA-256.

```go
buf := make([]byte, 32)
if _, err := rand.Read(buf); err != nil {
    return err
}
client := &http.Client{} // default transport verifies certificates
```

```go
id := strconv.FormatInt(time.Now().UnixNano(), 10) // predictable
client := &http.Client{Transport: &http.Transport{
    TLSClientConfig: &tls.Config{InsecureSkipVerify: true}, // #nosec not a fix
}}
```

What goes wrong: `math/rand` and timestamps are guessable. `InsecureSkipVerify` accepts any certificate, so the TLS peer is not authenticated.

## Do not decode `gob` from callers (A08)

`encoding/gob` and `encoding/xml` from a request body are the wrong parsers for untrusted data. Decode JSON into a struct you defined.

```go
var body TransferRequest
if err := json.NewDecoder(r.Body).Decode(&body); err != nil {
    http.Error(w, "bad request", http.StatusBadRequest)
    return
}
```

```go
dec := gob.NewDecoder(r.Body)
dec.Decode(&payload)
```

What goes wrong: gob restores Go values from a stream the caller built. A struct with fixed fields does not.

## Errors and logs (A09, A10)

Return a generic status to the client. Log the error server-side without the password, the token, or the PAN. If `json.NewDecoder` fails, respond 400 and do not continue with a zero struct that later looks authorized.

```go
log.Printf("login_failed user=%s", userID)
http.Error(w, "unauthorized", http.StatusUnauthorized)
```

```go
log.Printf("login %s %s", email, password)
http.Error(w, err.Error(), http.StatusInternalServerError)
```
