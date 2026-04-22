# Social Authentication Documentation

This document explains how Google, Apple, and Telegram login/signup works
in the ZedSMS app. It covers the full flow from the mobile app to the backend.

---

## Table of Contents

- [How It Works (Overview)](#how-it-works-overview)
- [Google Sign In](#google-sign-in)
- [Apple Sign In](#apple-sign-in)
- [Telegram Login](#telegram-login)
- [API Response](#api-response)
- [Important Notes](#important-notes)

---

## How It Works (Overview)

All three providers follow the same idea:

```
1. User taps "Login with Google/Apple/Telegram" in the app
2. Provider verifies the user's identity
3. Provider gives the app a token
4. App sends the token to our backend
5. Backend verifies the token
6. Backend finds or creates the user
7. Backend returns a Sanctum token
8. App uses the Sanctum token for all future requests
```

There is **no separate signup and login endpoint**.
The same endpoint handles both — if the user exists, they are logged in.
If not, a new account is created automatically.

---

## Google Sign In

### How Google works

Google's SDK gives the app an `id_token` (a JWT) directly.
The app sends this token to the backend. The backend verifies it
with Google's servers and logs in or creates the user.

### Flow

```
Mobile App
    │
    ├──1──► User taps "Sign in with Google"
    │
    ├──2──► Google SDK opens Google account picker
    │
    ├──3──► User selects their account
    │
    ◄──4──┘ Google SDK returns id_token
    │
    ├──5──► App sends id_token to backend
    │           POST /api/app/social/google
    │           { "id_token": "eyJhbGci..." }
    │
    │           Backend
    │               ├── Sends id_token to Google to verify
    │               ├── Gets user info (email, name, picture)
    │               ├── Finds or creates user in database
    │               └── Creates SIP credential if new user
    │
    ◄──6──┘ Backend returns Sanctum token + user data
```

### What Google provides

| Field | Description | Always available? |
|---|---|---|
| `sub` | Google's unique user ID | ✅ Yes |
| `email` | User's Gmail address | ✅ Yes |
| `name` | Full name | ✅ Yes |
| `picture` | Profile photo URL | ✅ Yes |

### API Endpoint

```
POST /api/app/social/google
Content-Type: application/json

{
    "id_token": "eyJhbGciOiJSUzI1NiIs..."
}
```

### Flutter Code

```dart
import 'package:google_sign_in/google_sign_in.dart';

final GoogleSignIn googleSignIn = GoogleSignIn(
    scopes: ['email', 'profile'],
);

final GoogleSignInAccount? account = await googleSignIn.signIn();
final GoogleSignInAuthentication auth = await account!.authentication;

// Send to backend
await dio.post('/api/app/social/google', data: {
    'id_token': auth.idToken,
});
```

---

## Apple Sign In

### How Apple works

Apple's SDK gives the app an `identity_token` (a JWT).
The app sends this token plus the user's name to the backend.
The backend verifies the JWT using Apple's public keys.

> **Important:** Apple only provides the user's name and email
> on the **very first login**. After that, only the unique user ID
> is returned inside the token. Our backend saves the name on first
> login so it is never lost.

> **Platform:** Apple Sign In is only available on iOS/Apple devices.
> It is not shown on Android.

### Flow

```
iPhone
    │
    ├──1──► User taps "Sign in with Apple"
    │
    ├──2──► Apple shows Face ID / Touch ID prompt
    │           (first time also asks to share email and name)
    │
    ◄──3──┘ Apple SDK returns identity_token + name (first login only)
    │
    ├──4──► App sends identity_token to backend
    │           POST /api/app/social/apple
    │           {
    │               "identity_token": "eyJhbGci...",
    │               "name": "John Doe"  ← only send if available
    │           }
    │
    │           Backend
    │               ├── Verifies JWT using Apple's public keys
    │               ├── Extracts user ID (sub) and email from token
    │               ├── Finds or creates user in database
    │               └── Creates SIP credential if new user
    │
    ◄──5──┘ Backend returns Sanctum token + user data
```

### What Apple provides

| Field | Description | Always available? |
|---|---|---|
| `sub` | Apple's unique user ID (inside JWT) | ✅ Yes |
| `email` | User's Apple email (inside JWT) | ❌ First login only |
| `givenName` | First name (from SDK) | ❌ First login only |
| `familyName` | Last name (from SDK) | ❌ First login only |
| `picture` | Profile photo | ❌ Never |

### API Endpoint

```
POST /api/app/social/apple
Content-Type: application/json

{
    "identity_token": "eyJhbGciOiJSUzI1NiIs...",
    "name": "John Doe"
}
```

> Send `name` as empty string `""` on repeat logins.
> The backend ignores it if the user already has a name saved.

### Flutter Code

```dart
import 'package:sign_in_with_apple/sign_in_with_apple.dart';

final credential = await SignInWithApple.getAppleIDCredential(
    scopes: [
        AppleIDAuthorizationScopes.email,
        AppleIDAuthorizationScopes.fullName,
    ],
);

// Build name safely (null on repeat logins)
final name = [credential.givenName, credential.familyName]
    .where((e) => e != null && e.isNotEmpty)
    .join(' ')
    .trim();

// Send to backend
await dio.post('/api/app/social/apple', data: {
    'identity_token': credential.identityToken,
    'name': name,
});
```

### Xcode Setup Required

1. Open `ios/Runner.xcworkspace` in Xcode
2. Click **Runner** → **Signing & Capabilities**
3. Click **+** Capability → add **Sign In with Apple**
4. Make sure Bundle ID matches: `com.zedsms.app`

---

## Telegram Login

### How Telegram works

Telegram uses OpenID Connect. The app opens Telegram's authorization
page in a browser. The user approves in their Telegram app.
Telegram redirects back with a `code`. The app sends this `code`
to the backend. The backend exchanges the code for an `id_token`
with Telegram's servers and logs in or creates the user.

> **Note:** Telegram never provides the user's email address.
> Telegram users will have `email: null` in the database.
> This is normal and expected.

### Flow

```
Mobile App
    │
    ├──1──► User taps "Login with Telegram"
    │
    ├──2──► App opens this URL in browser:
    │           https://oauth.telegram.org/auth
    │               ?client_id=6797774309
    │               &redirect_uri=YOUR_APP_DEEP_LINK
    │               &response_type=code
    │               &scope=openid profile
    │               &state=random_string
    │
    ├──3──► If Telegram app is installed:
    │           Telegram app opens and shows approval screen
    │           "Allow ZedSMS to access your account?"
    │           User taps ALLOW
    │
    │       If Telegram app is NOT installed:
    │           Browser opens Telegram web login
    │           User enters phone number and OTP
    │
    ◄──4──┘ Telegram redirects to app deep link with code:
    │           yourapp://auth?code=abc123&state=random_string
    │
    ├──5──► App verifies state matches, extracts code
    │
    ├──6──► App sends code to backend:
    │           POST /api/app/social/telegram
    │           { "code": "abc123" }
    │
    │           Backend
    │               ├── Exchanges code with Telegram for id_token
    │               ├── Verifies JWT using Telegram's public keys
    │               ├── Extracts name, photo, phone from token
    │               ├── Finds or creates user in database
    │               └── Creates SIP credential if new user
    │
    ◄──7──┘ Backend returns Sanctum token + user data
```

### What Telegram provides (inside JWT)

| Field | Description | Always available? |
|---|---|---|
| `sub` | Telegram's unique user ID | ✅ Yes |
| `name` | Full name | ✅ Yes |
| `picture` | Profile photo URL | ✅ Yes |
| `preferred_username` | Telegram username | ✅ If set |
| `phone_number` | Phone number | ✅ If phone scope added |
| `email` | Email address | ❌ Never |

### API Endpoint

```
POST /api/app/social/telegram
Content-Type: application/json

{
    "code": "abc123xyz"
}
```

> The `code` expires in **30 seconds**. Send it to the backend immediately
> after receiving it from the Telegram redirect.

### What is a Deep Link and Why Is It Required?

When Telegram finishes the login, it needs to redirect the user **back into your mobile app** with
the authorization `code`. A deep link is the mechanism that makes this possible.

A deep link is a custom URL scheme that your app registers with the phone's operating system.
For ZedSMS the scheme is `zedsms://`. When the phone sees any URL starting with `zedsms://`,
it knows to open the ZedSMS app instead of a browser.

**Without a deep link the flow breaks here:**

```
Telegram approves login → redirects to ??? → code is lost → user stuck in browser
```

**With a deep link the flow works end-to-end:**

```
Telegram approves login
    → redirects to zedsms://auth?code=abc123&state=xyz
    → phone OS sees "zedsms://" scheme
    → phone opens ZedSMS app and passes the URL to it
    → app reads code=abc123 from the URL
    → app sends code to backend
    → backend exchanges code for token → user logged in
```

**Production setup required (one time):**

| What | Where | Value |
|---|---|---|
| Register scheme on Android | `AndroidManifest.xml` | `android:scheme="zedsms"` |
| Register scheme on iOS | `Info.plist` | `CFBundleURLSchemes: zedsms` |
| Add redirect URI in BotFather | BotFather → Edit Bot → Redirect URIs | `zedsms://auth` |
| Set in backend `.env` | `TELEGRAM_OPENID_REDIRECT_URI` | `zedsms://auth` |

The value `zedsms://auth` must be **identical** in all four places. If it does not match,
Telegram will reject the request with `redirect_uri_mismatch`.

> **Testing note:** During backend testing without a real device, you can temporarily set
> `TELEGRAM_OPENID_REDIRECT_URI` to an HTTPS tunnel URL (like `https://app.exposr.dev/`) and
> manually copy the `code` from the browser. For production with a real app, always use
> the `zedsms://auth` deep link.

### Flutter Setup

**Android — `android/app/src/main/AndroidManifest.xml`**

Add this inside the `<activity>` tag:

```xml
<intent-filter>
    <action android:name="android.intent.action.VIEW" />
    <category android:name="android.intent.category.DEFAULT" />
    <category android:name="android.intent.category.BROWSABLE" />
    <data android:scheme="zedsms" android:host="auth" />
</intent-filter>
```

**iOS — `ios/Runner/Info.plist`**

```xml
<key>CFBundleURLTypes</key>
<array>
    <dict>
        <key>CFBundleURLSchemes</key>
        <array>
            <string>zedsms</string>
        </array>
    </dict>
</array>
```

### Flutter Code

```dart
import 'package:url_launcher/url_launcher.dart';
import 'package:uni_links/uni_links.dart'; // for deep link handling

// Step 1: Generate random state for security
final state = DateTime.now().millisecondsSinceEpoch.toString();

// Step 2: Open Telegram auth URL
// redirect_uri must match what is registered in BotFather and in backend .env
const redirectUri = 'zedsms://auth';

final url = Uri.parse(
    'https://oauth.telegram.org/auth'
    '?client_id=6797774309'
    '&redirect_uri=$redirectUri'
    '&response_type=code'
    '&scope=openid%20profile'
    '&state=$state'
);
await launchUrl(url, mode: LaunchMode.externalApplication);

// Step 3: Listen for deep link redirect
// Phone OS intercepts zedsms://auth?code=abc123&state=xyz and opens the app
final deepLink = await getInitialLink(); // zedsms://auth?code=abc123&state=...
final uri = Uri.parse(deepLink!);

// Step 4: Verify state matches (security check)
if (uri.queryParameters['state'] != state) {
    throw Exception('State mismatch — possible CSRF attack');
}

// Step 5: Extract code and send to backend
final code = uri.queryParameters['code'];
await dio.post('/api/app/social/telegram', data: {
    'code': code,
});
```

---

## API Response

All three providers return the **exact same response format**:

### Success (200)

```json
{
    "message": "Logged in",
    "data": {
        "token": "1|sanctumTokenHere...",
        "user": {
            "id": 1,
            "name": "John Doe",
            "email": "john@gmail.com",
            "phone": null,
            "email_verified_at": "2026-04-22T00:00:00.000000Z",
            "balance": "0.00",
            "status": 1,
            "created_at": "2026-04-22T00:00:00.000000Z",
            "updated_at": "2026-04-22T00:00:00.000000Z",
            "sip_credential": {
                "sip_username": "john_1",
                "sip_password": "abc123def",
                "status": "active"
            },
            "login_security": null
        }
    }
}
```

### Error Responses

| Status | Message | Reason |
|---|---|---|
| `401` | Invalid Google token | Token expired or invalid |
| `401` | Google token audience mismatch | Wrong app client ID |
| `401` | Invalid Apple token | Token expired or invalid |
| `401` | Failed to exchange Telegram code | Code expired (30 sec limit) |
| `401` | Invalid Telegram token | JWT verification failed |
| `403` | User disabled, please contact admin | Account is disabled |
| `422` | Validation error | Missing required field |

### Using the token

After login, store the token and send it in every request:

```dart
// Store token
await storage.write(key: 'token', value: response.data['data']['token']);

// Use in requests
dio.options.headers['Authorization'] = 'Bearer $token';
```

---

## Important Notes

### One endpoint for both signup and login

You do not need separate endpoints for signup and login.
The same endpoint handles both:

- **New user** → account created automatically → token returned
- **Existing user** → found in database → token returned
- **Existing email user** → accounts linked automatically → token returned

### Telegram users have no email

Telegram never provides an email address. This is normal.
Telegram users will have `email: null` in the database.
All app features work normally without an email.

### Apple name is only available once

Apple only sends the user's name on the very first login.
The mobile app must send the name to the backend on first login.
The backend saves it and never overwrites an existing name.

### Token expiry

Social tokens expire quickly — always send them to the backend immediately:

| Provider | Token expires in |
|---|---|
| Google | 1 hour |
| Apple | 10 minutes |
| Telegram | 30 seconds |

Your Sanctum token (returned by backend) does **not** expire
unless the user logs out.

### Security — Telegram state parameter

Always verify the `state` parameter when handling the Telegram redirect.
This prevents CSRF attacks. If the state does not match what you sent,
reject the login attempt.
