# Ruby

Rails helpers do not make string SQL, `html_safe`, or `Marshal` safe. Use the category that matches the sink.

## Queries use binds or hashes (A05)

Pass a hash to `where`, or a template with `?`. Do not interpolate into a SQL string. `find_by_sql` has the same rule.

```ruby
Ledger.where(id: id, owner_id: current_user.id)
```

```ruby
Ledger.where("id = '#{params[:id]}'")
```

What goes wrong: the interpolated string is SQL. A hash is bound by the adapter. `order(params[:sort])` is still SQL unless the column is taken from an allowlist.

## Views escape (A05)

Leave Rails auto-escape on. Call `html_safe` or `raw` only on markup your own code built. A user's note is not that markup.

```erb
<p><%= note %></p>
```

```erb
<p><%= raw note %></p>
```

What goes wrong: `raw`, `html_safe`, and `<%==` insert the string as HTML.

## Processes take separate arguments (A05)

Use the multi-argument form of `system`, `exec`, or `Open3.capture2`. Do not build one string, and do not use `Kernel#open` on a caller string. `open` with a leading pipe runs a process.

```ruby
system("ping", "-c", "1", host, exception: true)
```

```ruby
system("ping -c 1 #{host}")
output = `ping -c 1 #{host}`
open(params[:path])
```

What goes wrong: a single-string `system`, backticks, and `%x{}` use a shell. `Kernel#open` treats a pipe prefix as a command and a URL as a fetch.

## Paths stay under a root (A05)

Expand the root, join, expand again, and require the result to start with the root plus a separator. Store uploads under a generated name.

```ruby
root = File.expand_path(storage)
full = File.expand_path(name, root)
prefix = root.end_with?(File::SEPARATOR) ? root : root + File::SEPARATOR
raise "path" unless full.start_with?(prefix)
```

```ruby
File.read(File.join(storage, name))
```

What goes wrong: `File.join` will not keep `name` inside `storage` when `name` climbs or is absolute.

## Sessions, cookies, and passwords (A04, A07)

`has_secure_password` or bcrypt. `secret_key_base` comes from the environment, not the repo. Session cookies stay HttpOnly and Secure. `protect_from_forgery` stays on for cookie-authenticated browser forms. Filter passwords out of logs.

```ruby
# config/application.rb
config.filter_parameters += [:password, :token, :pan]
```

```ruby
Rails.logger.info("login #{email} #{password}")
```

```ruby
cookies[:sid] = { value: sid, httponly: true, secure: true, same_site: :lax }
```

```ruby
cookies[:sid] = sid
```

What goes wrong: a cookie without HttpOnly is readable by script. A password in the log line is copied into every log store. A committed `secret_key_base` lets anyone forge a cookie.

## YAML and Marshal (A08)

Load untrusted YAML with `YAML.safe_load` and an explicit permitted class list. Do not `Marshal.load` a cookie, a cache value, or a request body. JSON into a hash you then permit-list is the boundary format.

```ruby
data = YAML.safe_load(text, permitted_classes: [], aliases: false)
```

```ruby
data = YAML.load(text)       # Psych still builds arbitrary types on older calls
data = Marshal.load(cookie)
```

What goes wrong: `Marshal` and an unrestricted YAML load construct Ruby objects the payload names. `safe_load` with an empty class list yields basic types only.

## Do not call a method the caller names (A01)

Dispatch with an allowlist. `send`, `public_send`, and `constantize` on `params` let the caller pick the method or class.

```ruby
case params[:action_name]
when "archive" then archive
when "restore" then restore
else raise ActionController::BadRequest
end
```

```ruby
send(params[:method], params[:arg])
```

What goes wrong: the request chooses which method runs, including methods that were never linked from the UI.
