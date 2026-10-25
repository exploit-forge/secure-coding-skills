# JavaScript

Browser and Node. TypeScript query and cookie examples live in the category pages. This page is the JavaScript sinks those pages do not name: `innerHTML`, `eval`, `child_process`, and `path.join`. The categories are the same: HTML and shell and paths are A05, tokens are A07, unsafe object merge is A08.

## Put text in the DOM as text (A05)

Assign untrusted strings with `textContent`, or a framework binding that escapes. Build URLs with `URL` and an allowlisted origin.

```javascript
noteElement.textContent = note;
```

```javascript
noteElement.innerHTML = note;
document.write(note);
```

What goes wrong: `innerHTML`, `outerHTML`, `insertAdjacentHTML`, and `document.write` parse the string as HTML. A note saved earlier then runs for the next reader. `element.setAttribute("href", note)` is the same class when the value is a script URL.

## Do not evaluate strings (A05)

Call a function you wrote. Do not pass caller text to `eval`, `new Function`, or `setTimeout` / `setInterval` as a string.

```javascript
const fns = { refresh: refreshInbox };
const fn = fns[action];
if (fn) fn();
```

```javascript
eval(action);
setTimeout(action, 0);
```

What goes wrong: the string is code. A map of known actions keeps the caller on a list you control.

## Node starts processes with an argument array (A05)

Use `execFile` or `spawn` with a fixed command and an args array. `exec` runs a shell. Check a host against a hostname pattern before you pass it.

```javascript
import { execFile } from "node:child_process";

execFile("ping", ["-c", "1", host], { shell: false }, callback);
```

```javascript
import { exec } from "node:child_process";

exec("ping -c 1 " + host, callback);
```

What goes wrong: `exec`, `execSync`, and `spawn` with `shell: true` parse metacharacters. `shell: false` and a separate argument list do not.

## `path.join` does not contain a path (A05)

Resolve the root, reject an absolute name, and reject a relative path that climbs out. `path.join` and `path.normalize` alone still leave the root when a later segment is absolute, and they do not compare the result to the root.

```javascript
import path from "node:path";

const root = path.resolve(storage);
if (path.isAbsolute(name)) throw new Error("path");
const full = path.resolve(root, name);
const rel = path.relative(root, full);
if (rel === ".." || rel.startsWith(".." + path.sep) || path.isAbsolute(rel)) {
  throw new Error("path");
}
```

```javascript
const full = path.join(storage, name);
await fs.readFile(full);
```

What goes wrong: a name that climbs, or an absolute name on some joins, reads outside the directory you meant to expose.

## Do not merge caller keys onto prototypes (A08)

Copy known fields. Reject `__proto__`, `prototype`, and `constructor` as keys in a recursive merge. A `JSON.parse` of an object into a fresh variable is fine; a deep assign onto `{}` or onto a shared config is not.

```javascript
function pick(body) {
  return { theme: body.theme, label: body.label };
}
```

```javascript
function merge(target, body) {
  for (const key of Object.keys(body)) target[key] = body[key];
  return target;
}
```

What goes wrong: a key the caller chooses can change `Object.prototype` when the merge walks nested objects. Later code then reads a property it thinks is missing and gets the caller's value.

## Browser messages name the sender (A01)

Handle `postMessage` only when `event.origin` is on a fixed list. Send with a specific origin, not `*`, when the payload is private.

```javascript
const allowed = new Set(["https://app.example.com"]);
window.addEventListener("message", (event) => {
  if (!allowed.has(event.origin)) return;
  applySettings(event.data);
});
```

```javascript
window.addEventListener("message", (event) => {
  applySettings(event.data);
});
```

What goes wrong: any frame can post a message. Without an origin check the handler treats it as the app.

## Node SQL and logs

Bind placeholders (`?` or `$1`, matching the driver). Do not log passwords or tokens. Compare MACs with `crypto.timingSafeEqual` on equal-length buffers, as in the authentication page.
