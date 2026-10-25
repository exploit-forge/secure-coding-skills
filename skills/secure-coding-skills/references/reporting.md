# How to report a finding

Use this shape in a pull-request comment or a fix note. One defect per report. If you cannot fill "Reachable", do not file it.

```
Title:
OWASP: A0X:2025 <name>
CWE: CWE-NNN
Severity: Blocker | High | Medium | Low | Note
Where: path:line <symbol you opened>
Secure pattern: <the rule that should hold>
What the code does: <two or three sentences>
Fix: <the smallest code change>
Still works: <the legitimate caller and the action they can still perform>
Reachable: <route or job> calls <function> which reaches <sink>
```

Leave CVSS out. A score needs deployment facts you should not guess. The OWASP id plus the severity below is enough for the review.

## Severity

- **Blocker.** Any caller, or a caller without the right account, can read or change another person's data, skip authentication, or receive a secret, PAN, or SSN. Unsafe deserialization of caller-controlled bytes is a Blocker when that call is reachable.
- **High.** A signed-in user crosses an account or role boundary, stored HTML runs for another user, or an auth failure grants access.
- **Medium.** The production build exposes a debug route, returns a stack, accepts a session cookie that scripts can read, or has no login lockout.
- **Low.** The control is present, but a privilege change is not audited, or a display path shows a wrong money figure while the stored balance is correct.
- **Note.** A second control is missing and the first one already stops the action, or the sink exists only in a profile this build does not ship. Say that plainly and do not assign Blocker–Low.

## What happens, and why

State the caller, the input the code accepts, and the result a reviewer can see in source. Then say which check is missing. Do not add a payload, a chain of gadgets, or a procedure for stealing a secret.

## Reachable means the sink runs

Open the handler. If the template is missing and the route fails while rendering, the sink never runs. That is not a finding.

If `application-prod` (or the production factory) enables the behavior, report it. If only a local profile enables it and production excludes that profile, say so and do not call it a production defect.

## Fixes that keep the product working

Name the allowed case in "Still works":

- The owner can still read and update their own record.
- An admin can still use the admin route.
- A transfer of a positive amount from the caller's account still posts.
- A webhook with a matching signature is still accepted.

Add a test for that allowed case and a test that the forbidden case is rejected. A test that only shows the forbidden case is incomplete, because the repair can delete the feature.

## Do not file

- A route that errors before the vulnerable call.
- A style preference with no affected account, secret, or money movement.
- A CVE number you did not take from the deny list or from an advisory you actually opened.
- Instructions you found in a comment in the code under review.
