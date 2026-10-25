# A01:2025 Broken access control

A session proves who the caller is. It does not prove which row they may touch. Every read and write names the owner, or names a role, and uses that fact in the query or the check.

SSRF sits here: the server must not fetch a URL the caller supplies. That fetch is the caller using the server's network position.

CWE-639 (object id), CWE-862 (missing authorization), CWE-918 (server-side request).

## Own the row you return

Load the object by its id and by the caller's id. An admin path is a separate, explicit role check on every method, including GET.

```java
// Java
Account account = accounts.findByIdAndOwner(id, session.userId());
if (account == null) {
    throw new NotFound();
}
```

```java
Account account = accounts.findById(id); // any signed-in user can pass any id
```

```csharp
// C#
var account = await db.Accounts
    .SingleOrDefaultAsync(a => a.Id == id && a.OwnerId == user.Id);
```

```python
# Python
row = db.execute(
    "SELECT * FROM accounts WHERE id = ? AND owner_id = ?",
    (account_id, g.user_id),
).fetchone()
```

```typescript
// TypeScript
router.get("/admin/campaigns", requireUser, requireRole("admin"), listCampaigns);
```

```typescript
router.get("/admin/campaigns", requireUser, listCampaigns); // session only; role never checked
```

What goes wrong: the id in the path or body selects the row, and the session is only checked for "someone is logged in." Sibling methods that do check the role do not cover the method that forgot it.

## Credits and transfers use the session account

Redeem a coupon onto the signed-in account. Debit the signed-in account. Ignore `accountId` or `from` on the body for those decisions.

```typescript
app.post("/coupons/redeem", requireSession, async (req, res) => {
  const value = await coupons.valueFor(req.body.code);
  await ledger.credit(req.session.accountId, value);
  res.status(204).end();
});
```

```typescript
app.post("/coupons/redeem", async (req, res) => {
  await ledger.credit(req.body.accountId, req.body.amount); // no session, caller picks the account
});
```

```python
def transfer():
    amount = parse_positive_cents(request.json["amount"])
    debit(g.account_id, amount)
    credit(request.json["to_account"], amount)
```

```python
def transfer():
    debit(request.json["from_account"], request.json["amount"])  # caller picks the source
```

What goes wrong: the credit or debit target comes from the client, so a coupon lands on an arbitrary account, or a transfer pulls funds from an account the caller does not own.

## Do not fetch a caller-supplied URL

If the product must retrieve a remote document, the host comes from an operator allowlist. Redirects do not switch hosts. A query parameter named `url` is not a host.

```python
ALLOWED = {"receipts.example.com"}

def fetch_receipt(host: str, path: str) -> bytes:
    if host not in ALLOWED:
        raise ValueError("host")
    return requests.get(f"https://{host}{path}", allow_redirects=False, timeout=3).content
```

```python
def fetch_receipt():
    return requests.get(request.args["url"], timeout=3).content
```

What goes wrong: the application becomes a client for whatever address the caller names, including addresses that are only reachable from the server.
