# React Native

Treat the JavaScript bundle as public. The secure pattern is platform secret storage, an allowlisted WebView, HTTPS, and an API that still checks the owner. Browser DOM rules in the JavaScript page apply to React Native Web and to any WebView.

## Tokens live in the keychain (A04, A07)

Store session and refresh tokens in the iOS keychain or Android keystore (`expo-secure-store`, `react-native-keychain`, or the platform API). `AsyncStorage` is unencrypted app storage. A key shipped in the bundle is not a secret.

```javascript
import * as SecureStore from "expo-secure-store";

await SecureStore.setItemAsync("refresh", token);
```

```javascript
import AsyncStorage from "@react-native-async-storage/async-storage";

await AsyncStorage.setItem("refresh", token);
```

What goes wrong: `AsyncStorage` is readable from a device backup and from many debug tools. The server must still reject a token that fails signature checks.

## WebViews and deep links name a host (A01, A05)

Set `originWhitelist` to the hosts you render. Do not turn on `javaScriptEnabled` for an arbitrary `uri`. Handle `Linking` URLs by checking scheme and host, then navigate. The account comes from the session, not from a query parameter.

```javascript
<WebView
  source={{ uri: "https://app.example.com/help" }}
  originWhitelist={["https://app.example.com"]}
  onShouldStartLoadWithRequest={(req) => {
    const url = new URL(req.url);
    return url.host === "app.example.com";
  }}
/>
```

```javascript
<WebView source={{ uri: route.params.url }} javaScriptEnabled />
```

```javascript
function onUrl(url) {
  const parsed = new URL(url);
  if (parsed.protocol !== "myapp:" || parsed.hostname !== "pay") return;
  openTransfer(session.accountId);
}
```

```javascript
function onUrl(url) {
  const parsed = new URL(url);
  openTransfer(parsed.searchParams.get("account"));
}
```

What goes wrong: a link or a WebView URL chosen by another app can load script beside a bridge, or can point the UI at an account id the API might trust. The API owner check is still required.

## Release builds use HTTPS and do not log secrets (A02, A09)

Leave Android `usesCleartextTraffic` off. Do not set iOS `NSAllowsArbitraryLoads`. Do not `console.log` a password, a token, or a PAN. Do not `eval` a string from a push payload or a deep link.

```javascript
console.log("login failed", userId);
```

```javascript
console.log("login", email, password, token);
eval(payload.script);
```

What goes wrong: Metro and device logs retain the arguments. `eval` runs the string as JavaScript inside the app.

## Bridge methods are an allowlist (A01)

A native module exposes named methods. Do not pass a method name from JavaScript straight into a dynamic call on the native side, and do not let the bridge read the whole keychain. Each method checks the signed-in user before it returns data.
