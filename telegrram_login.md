# Telegram OpenID Connect (OIDC) Authentication Flow

## Overview

Telegram uses the standard OpenID Connect authorization code flow.
The mobile app gets a `code` from Telegram, sends it to the backend,
and the backend exchanges it for an `id_token` (JWT) to authenticate the user.

---

## Endpoints

| Purpose | URL |
|---|---|
| Authorization | `https://oauth.telegram.org/auth` |
| Token Exchange | `https://oauth.telegram.org/auth/token` |
| Public Keys (JWKS) | `https://oauth.telegram.org/keys` |
| Discovery | `https://oauth.telegram.org/.well-known/openid-configuration` |

---

## Credentials (from BotFather)

| Key | Value |
|---|---|
| Client ID | `6797774309` |
| Client Secret | stored in `.env` as `TELEGRAM_OPENID_CLIENT_SECRET` |
| Redirect URI | `https://app6797774309-login.tg.dev/tglogin` |

---

## Complete Flow

### Step 1 — App opens Telegram authorization URL

The mobile app opens this URL in a browser or webview:

```
https://oauth.telegram.org/auth
  ?client_id=6797774309
  &redirect_uri=https://app6797774309-login.tg.dev/tglogin
  &response_type=code
  &scope=openid profile
  &state=random_string_for_security
```

**Parameters:**

| Parameter | Description |
|---|---|
| `client_id` | Your Telegram app ID from BotFather |
| `redirect_uri` | Must exactly match what is registered in BotFather |
| `response_type` | Always `code` |
| `scope` | `openid profile` — Telegram does not provide email |
| `state` | Random string — must be verified on redirect to prevent CSRF |

---

### Step 2 — User logs in on Telegram

Telegram shows a login screen. The user authorizes your app.

---

### Step 3 — Telegram redirects back to the app

After authorization Telegram redirects to your registered URI with the code:

```
https://app6797774309-login.tg.dev/tglogin
  ?code=abc123xyz
  &state=same_random_string
```

The mobile app catches this deep link redirect and extracts the `code`.

> **Important:** Always verify that the `state` value matches what you sent in Step 1.

---

### Step 4 — App sends `code` to your backend

```http
POST /api/app/social/telegram
Content-Type: application/json

{
    "code": "abc123xyz"
}
```

---

### Step 5 — Backend exchanges `code` for `id_token`

The backend makes a server-to-server request to Telegram:

```http
POST https://oauth.telegram.org/auth/token
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code
&code=abc123xyz
&client_id=6797774309
&client_secret=your_secret
&redirect_uri=https://app6797774309-login.tg.dev/tglogin
```

Telegram responds with:

```json
{
    "access_token": "...",
    "id_token": "eyJhbGci...",
    "token_type": "Bearer",
    "expires_in": 3600
}
```

---

### Step 6 — Backend verifies the `id_token` JWT

The backend fetches Telegram's public keys from `https://oauth.telegram.org/keys`
and verifies the JWT signature. It then validates:

| Claim | Expected Value |
|---|---|
| `iss` | `https://oauth.telegram.org` |
| `aud` | `6797774309` (your client_id) |
| `exp` | must be in the future |

Decoded JWT claims from Telegram:

```json
{
    "sub": "123456789",
    "first_name": "John",
    "last_name": "Doe",
    "username": "johndoe",
    "photo_url": "https://..."
}
```

> **Note:** Telegram never provides `email` in the token.
> Users registered via Telegram will have `email = null` in the database.

---

### Step 7 — Backend returns Sanctum token to the app

```json
{
    "message": "Logged in",
    "data": {
        "token": "1|sanctum_token...",
        "user": {
            "id": 1,
            "name": "John Doe",
            "email": null,
            "email_verified_at": "2026-04-21T00:00:00.000000Z",
            "balance": "0.00",
            "status": 1,
            "sip_credential": {
                "sip_username": "telegram1_1",
                "sip_password": "abc123def",
                "status": "active"
            },
            "login_security": null
        }
    }
}
```

---

## Full Flow Diagram

```
Mobile App
    │
    ├──1──► Opens oauth.telegram.org/auth?client_id=6797774309&...
    │
    │            [ User sees Telegram login screen ]
    │            [ User authorizes the app        ]
    │
    ◄──2──┘  Redirected to deep link with ?code=abc123&state=xyz
    │
    │        App verifies state matches, extracts code
    │
    ├──3──► POST /api/app/social/telegram
    │            { "code": "abc123" }
    │
    │            Your Backend
    │                │
    │                ├──4──► POST oauth.telegram.org/auth/token
    │                │            { code, client_id, client_secret, ... }
    │                │
    │                ◄──5──┘  { id_token: "eyJhbGci..." }
    │                │
    │                └──6──   Verify JWT using Telegram's JWKS
    │                         Extract sub, first_name, last_name
    │                         Find or create user in database
    │                         Create SIP credential if new user
    │
    ◄──7──┘  { token: "sanctum...", user: { ... } }
    │
    App stores token and navigates to home screen
```

---

## Mobile Developer Responsibilities

| Step | Task |
|---|---|
| 1 | Build the Telegram auth URL and open it in browser/webview |
| 2 | Handle the deep link redirect and extract `code` |
| 2 | Verify `state` matches what was sent |
| 3 | Send `code` to `POST /api/app/social/telegram` |
| 4 | Store the returned Sanctum token securely |
| 5 | Use token in `Authorization: Bearer {token}` header for all future requests |

---

## Backend Responsibilities

| Step | Task |
|---|---|
| 5 | Exchange `code` for `id_token` at Telegram's token endpoint |
| 6 | Verify JWT signature using Telegram's JWKS public keys |
| 6 | Validate `iss`, `aud`, `exp` claims |
| 6 | Find or create user by `sub` (Telegram user ID) |
| 6 | Create SIP credential for new users |
| 7 | Return Sanctum token |

---

## Database Result

### `users` table
| Field | Value |
|---|---|
| `name` | `John Doe` |
| `email` | `null` |
| `email_verified_at` | set to `now()` automatically |
| `password` | random hash (user never uses it) |

### `providers` table
| Field | Value |
|---|---|
| `provider` | `telegram` |
| `provider_id` | Telegram `sub` e.g. `123456789` |
| `name` | `John Doe` |
| `avatar` | Telegram photo URL |

---

## Error Responses

| Status | Message | Reason |
|---|---|---|
| `401` | Failed to exchange Telegram code | Invalid or expired code |
| `401` | Invalid Telegram token | JWT verification failed |
| `403` | User disabled, please contact admin | User account is disabled |
