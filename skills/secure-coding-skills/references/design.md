# A06:2025 Insecure design

Some bugs are the workflow. The code does what was written, and what was written credits the wrong person or resets the wrong account. Design the allowed transition, then reject everything else before you touch a balance.

Also treat a formatter or a column lookup as security-relevant when it changes what money the user sees or what amount you post. A checksum or amount parser that accepts a bad value is in scope when a payment or an allow decision uses it.

CWE-840 (business logic), CWE-640 (weak password recovery).

## Money is integer minor units

Store cents (or the currency's minor unit) as an integer. Convert to major units in one display function. APIs that move money accept and return minor units. Do not add, subtract, or compare floating major units.

```java
public final class Money {
    public static String major(long cents) {
        return BigDecimal.valueOf(cents).movePointLeft(2).toPlainString();
    }
}
```

```java
double shown = cents / 100.0; // scattered through pages, easy to mix with major units
```

```typescript
export function majorUnits(cents: number): string {
  const sign = cents < 0 ? "-" : "";
  const abs = Math.abs(cents);
  return `${sign}${Math.trunc(abs / 100)}.${String(abs % 100).padStart(2, "0")}`;
}
```

What goes wrong: one screen divides and another already received major units, so the ledger and the page disagree. Keeping a single formatter makes that visible in review.

## Read the column name the database stored

H2 folds an unquoted identifier to uppercase, so a column written as `amount_cents` is stored as `AMOUNT_CENTS`. Some row maps keep that folded key. A history page that looks up `amount_cents` gets nothing and prints `0.0`, while the balance path that reads `AMOUNT_CENTS` stays correct.

Quote the name when you create the column and when you read it, or look up the folded key on purpose. Test the read against the dialect you ship. A mapper that works on Postgres can still hide a zero on H2.

```java
Map<String, Object> row = jdbc.queryForMap(
    "SELECT \"amount_cents\" FROM ledger WHERE id = ?", id);
long cents = ((Number) row.get("amount_cents")).longValue();
return Money.major(cents);
```

```java
Map<String, Object> row = jdbc.queryForMap(
    "SELECT amount_cents FROM ledger WHERE id = ?", id);
Object raw = row.get("amount_cents"); // map key is AMOUNT_CENTS, so raw is null
return Money.major(raw == null ? 0 : ((Number) raw).longValue()); // history shows 0.0
```

```sql
CREATE TABLE ledger (
  id VARCHAR(36) PRIMARY KEY,
  "amount_cents" BIGINT NOT NULL
);
```

What goes wrong: the two screens do not use the same key, so one balance is right and the other is a silent zero. That is a money bug. Any engine that folds case has the same rule: the string in application code must be the key the driver actually returns.

## Reject a non-positive amount before you change a balance

Decide the sign in the operation (debit or credit), and require the magnitude to be a positive integer of minor units. Do not run `abs` and then subtract the original signed value. Do both legs in one transaction.

```python
def transfer(sender: str, recipient: str, cents: int) -> None:
    if cents <= 0:
        raise ValueError("amount")
    with db.transaction():
        debit(sender, cents)
        credit(recipient, cents)
```

```python
def transfer(sender: str, recipient: str, amount: int) -> None:
    if abs(amount) > 0:
        debit(sender, amount)  # a negative amount increases the sender
```

What goes wrong: `abs(amount) > 0` is true for a negative amount. Subtracting that negative value credits the sender. The check looks like validation and does the opposite of a transfer.

## A reset token names the account

Consume a single-use token, read the user id from that record, and set the password there. A user id in the body is ignored. Expire the token when it is used, and expire other sessions for that user.

```java
ResetToken token = resets.consume(body.token());
users.setPassword(token.userId(), passwordHasher.hash(body.newPassword()));
```

```java
resets.validate(body.userId(), body.token());
users.setPassword(body.userId(), passwordHasher.hash(body.newPassword()));
```

What goes wrong: the body selects the account, so the token and the account are not required to be the same row the caller should be allowed to change. The safe handler never takes the user id from the client.
