# Rust

The borrow checker does not check SQL, HTML, shell strings, or paths. Apply the same categories as the other language pages.

## Queries bind (A05)

Use the driver's bind API (`sqlx` `$1` / `?`, `rusqlite` `params!`). Do not `format!` a value into the statement. Allowlist any column or sort direction.

```rust
sqlx::query("SELECT id FROM ledger WHERE id = $1 AND owner_id = $2")
    .bind(&id)
    .bind(&owner_id)
    .fetch_optional(&pool)
    .await?;
```

```rust
let sql = format!("SELECT id FROM ledger WHERE id = '{id}'");
sqlx::query(&sql).fetch_optional(&pool).await?;
```

What goes wrong: the formatted string is parsed as SQL. Bind placeholders are not.

## Commands are separate arguments (A05)

`std::process::Command` does not invoke a shell unless you ask for one. Pass the host as one argument after you check it.

```rust
if !hostname(host) {
    return Err(Error::BadHost);
}
Command::new("ping").arg("-c").arg("1").arg(host).status()?;
```

```rust
Command::new("sh").arg("-c").arg(format!("ping -c 1 {host}")).status()?;
```

What goes wrong: `sh -c` parses the whole string. Argument methods do not.

## Paths stay under a root (A05)

Reject absolute untrusted names. Join, then `canonicalize` when the file exists, and require `starts_with` on the canonical root. For a file you are about to create, walk `components` and reject `ParentDir` instead of canonicalizing a missing path.

```rust
let root = root.canonicalize()?;
let full = root.join(name).canonicalize()?;
if !full.starts_with(&root) {
    return Err(Error::Path);
}
```

```rust
let full = root.join(name);
std::fs::read(&full)?;
```

What goes wrong: `join` will still attach `..` segments, and `read` follows them. `starts_with` on a non-canonical path is not enough, because a symlink or a `..` can leave the root after the OS resolves it.

## Do not turn off certificate checks (A04, A07)

Use the default verifier in `rustls` or `reqwest`. Leave `danger_accept_invalid_certs` unset. Draw tokens from the operating system RNG (`getrandom` or `rand::rngs::OsRng`), not a seeded `StdRng` with a constant seed. Hash passwords with argon2 or bcrypt.

```rust
let client = reqwest::Client::builder().build()?;
let mut buf = [0u8; 32];
getrandom::fill(&mut buf)?;
```

```rust
let client = reqwest::Client::builder()
    .danger_accept_invalid_certs(true)
    .build()?;
```

What goes wrong: accepting any certificate authenticates nobody. A seeded RNG makes the next token predictable.

## Decode into a struct (A08)

`serde_json` into a type you wrote. Do not `serde_yaml::from_str` into `serde_yaml::Value` and then interpret type tags, and do not accept `bincode` or `rmp_serde` from the network unless both sides are the same versioned schema you control. Prefer JSON for a boundary you do not fully control.

```rust
let body: TransferRequest = serde_json::from_slice(&bytes)?;
```

```rust
let body: serde_json::Value = serde_json::from_slice(&bytes)?;
// later: type name taken from body["type"] and instantiated
```

What goes wrong: a free-form value lets the payload choose which fields exist and, if you dispatch on a type string, which code runs. A struct rejects unknown shapes at the boundary.

## Logs omit secrets (A09)

Do not `Debug`-print a struct that contains a password, token, or PAN. Redact those fields or keep them out of the type you log. Map a parse error to 400 and an auth error to 401 without sending `Display` of an internal error to the client.

```rust
tracing::info!(user_id = %user_id, "login_failed");
```

```rust
tracing::info!(?password, "login");
```
