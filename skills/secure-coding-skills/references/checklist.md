# Review checklist

Run this on a pull request or before you call new code done. Stop on the first failing item that the diff actually touches, fix that pattern, then continue. Skip a line when the change does not have that surface.

## Before you file anything

- [ ] You opened the file and can name the function that runs.
- [ ] The dangerous call is reachable from a route, job, or parser that this build can hit.
- [ ] A missing view, dead branch, or earlier exception is not being reported as a finding.
- [ ] The write-up names who is still allowed to succeed.

## Access (A01:2025)

- [ ] The handler loads the row for this caller, or checks an admin role on every method, including reads.
- [ ] A coupon, credit, or transfer uses the session account. The body does not choose the account.
- [ ] The server does not request a URL the caller supplied.

## Configuration (A02:2025)

- [ ] Clients receive a generic error and a request id. Stack traces stay in server logs.
- [ ] Debug and environment routes are absent from the production build.
- [ ] CORS allows a fixed origin list. Credentials are not paired with a reflected or wildcard origin.
- [ ] The user id and role come from the session, not from a client header.
- [ ] An embedded database console is off in shared environments, and it does not use an empty password.
- [ ] Pages and JSON do not print a team secret, a connection string, or an internal hostname unless that screen is specified to show it.

## Dependencies (A03:2025)

- [ ] The lockfile pins an exact version. A comment, classifier, or the word "lab-weak" is not the pin.
- [ ] Resolved versions are outside the deny list in `dependencies.md`.
- [ ] You did not invent a CVE for a package that is not on that list.

## Cryptography (A04:2025)

- [ ] Passwords use a password hash (Argon2id or bcrypt).
- [ ] Encryption is an authenticated mode. The key is not a literal in source, and ECB is not used.
- [ ] Responses do not include a full PAN or SSN beside ciphertext.

## Injection (A05:2025)

- [ ] SQL uses placeholders for the driver's API.
- [ ] User text is escaped. `th:utext` and `Html.Raw` are not used for it.
- [ ] Process arguments are a list. The host or path is not concatenated into a shell string.
- [ ] File paths are resolved under a root, and a path that escapes the root is rejected.
- [ ] Uploads are stored under a generated name. The client filename is not the path.

## Design and money (A06:2025)

- [ ] Amounts are stored as integer minor units. Major units are formatted in one place.
- [ ] The column name you read is the name the database stored (`AMOUNT_CENTS` vs `amount_cents`).
- [ ] Amounts less than or equal to zero are rejected before `abs` or subtraction.
- [ ] A password reset updates the user named by the token, not a user id from the body.

## Authentication (A07:2025)

- [ ] Tokens without a signature segment are rejected.
- [ ] The full MAC is compared in constant time. A short prefix is not enough.
- [ ] The MAC is an HMAC. The code does not compare a hash of `secret + body`.
- [ ] The session cookie is HttpOnly (and Secure on HTTPS).
- [ ] Login has a lockout or rate limit, and the password is not written to logs.
- [ ] A parse error denies the request.

## Integrity (A08:2025)

- [ ] Untrusted YAML uses a safe loader. SnakeYAML global tags stay off.
- [ ] `BinaryFormatter` and `ObjectInputStream` are not used on caller bytes.
- [ ] Webhooks verify the provider signature over the raw body and reject a missing signature.

## Logging and failures (A09:2025, A10:2025)

- [ ] Role changes, resets, and money movement write an audit record with actor, action, and target.
- [ ] Logs omit passwords, PANs, and SSNs.
- [ ] If the auth dependency is down, the privileged action is denied.
- [ ] Bad input, including a zero divisor, returns a client error without a stack trace.

## Languages

Open the matching file under `references/languages/` when the diff is PHP, JavaScript, Go, Rust, Ruby, C#, Flutter, or React Native. The category checks above still apply. Also confirm the language-specific sink: PHP `unserialize` and `shell_exec`, JavaScript `innerHTML` / `eval` / `child_process` / `path.join`, Go `sh -c` and `InsecureSkipVerify`, Rust `sh -c` and `danger_accept_invalid_certs`, Ruby `Marshal` and `html_safe`, C# `BinaryFormatter` and `FromSqlRaw`, Flutter and React Native token storage and WebView origin.

## Infrastructure (A02, A03, A04)

Skip this block when the diff has no Terraform, Bicep, or CloudFormation.

- [ ] Data storage is private. Public ACLs, public blob access, and `PublicRead` are off unless the bucket is an intentional public site.
- [ ] Security groups and NSGs do not allow `0.0.0.0/0`, `::/0`, or `*` onto administration or database ports.
- [ ] IAM and role definitions name specific actions and resources. `*` on action or resource is not the app role.
- [ ] Secrets are not committed in tfvars, state, Bicep defaults, or CloudFormation parameter defaults. `sensitive` and `NoEcho` do not replace a secret store.
- [ ] Azure apps that call Azure APIs use a managed identity instead of a password in app settings.
