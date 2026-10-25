# Flutter (Dart)

A Flutter app is a client. Tokens, WebViews, and cleartext traffic are the defects that show up in review. Server rules in the category pages still apply to any Dart backend. Categories: storage and TLS are A02 and A04, WebViews and paths are A05, deep links that select an account are A01, logs are A09.

## Keep session material in the platform store (A04, A07)

Put refresh tokens and session secrets in the platform keystore (`flutter_secure_storage` or the equivalent). `SharedPreferences` is a plain file. An API key compiled into the binary is public; the server still has to authorize the signed-in user.

```dart
await storage.write(key: 'refresh', value: token);
```

```dart
await prefs.setString('refresh', token);
```

What goes wrong: a backup, a rooted device, or a debug snapshot copies `SharedPreferences`. The secure store is still not a substitute for a short-lived token and a server-side owner check.

## WebViews run only first-party content (A05)

Enable JavaScript only for an origin you control. Block navigation that leaves that origin. Do not add a JavaScript channel that can move money or read the token unless the page is yours and the channel checks the call.

```dart
final controller = WebViewController()
  ..setJavaScriptMode(JavaScriptMode.unrestricted)
  ..setNavigationDelegate(NavigationDelegate(
    onNavigationRequest: (request) {
      final uri = Uri.parse(request.url);
      if (uri.host != 'app.example.com' || uri.scheme != 'https') {
        return NavigationDecision.prevent;
      }
      return NavigationDecision.navigate;
    },
  ));
```

```dart
WebViewController()
  ..setJavaScriptMode(JavaScriptMode.unrestricted)
  ..loadRequest(Uri.parse(userSuppliedUrl));
```

What goes wrong: a caller-supplied URL with JavaScript on runs in the app's WebView, next to any channel you exposed.

## Do not allow cleartext, and do not print secrets (A02, A09)

Release builds talk HTTPS only. Android `usesCleartextTraffic` stays false. iOS does not add a blanket ATS exception. `debugPrint` and `log` omit tokens, passwords, and PANs. `print` of a response body is the same leak.

```dart
debugPrint('login failed user=$userId');
```

```dart
debugPrint('login $email $password token=$token');
```

What goes wrong: device logs and the Flutter console keep whatever you print. A cleartext exception sends the session on a network the user does not control.

## Paths and accounts (A05, A01)

Join a file name under the app support directory and reject a name that climbs out. A deep link may open a screen; the account id still comes from the signed-in session, not from a query parameter on the link.

```dart
final root = Directory(supportPath);
final full = path.normalize(path.join(root.path, name));
if (!path.isWithin(root.path, full)) {
  throw StateError('path');
}
```

```dart
File(path.join(supportPath, name)).readAsString();
final accountId = uri.queryParameters['account'];
```

What goes wrong: `join` does not contain a path. A link that chooses the account id shows another person's data if the API trusts that parameter. The API must apply the owner check in the access page.
