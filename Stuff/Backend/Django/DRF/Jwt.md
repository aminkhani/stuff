```table-of-contents
```
## 🔑 Authentication vs Authorization

- **Authentication** 🪪 → _"Who are you?"_ Verifying identity.
- **Authorization** 🚦 → _"What can you do?"_ Verifying permissions/scope.

DRF separates these cleanly:

- `authentication_classes` → identifies the user
- `permission_classes` → decides what they can access

---
## 🧰 Authentication Methods in DRF

### 1. 🍪 Session Authentication

Uses Django's built-in session framework + cookies.

```python
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework.authentication.SessionAuthentication',
    ]
}
```

**✅ Pros**

- Built into Django, zero extra setup
- CSRF protection out of the box
- Great for browsable API / same-origin web apps

**❌ Cons**

- Requires cookies → not great for mobile/native apps
- Stateful (server must store session) → harder to scale horizontally
- Vulnerable to CSRF if misconfigured (needs `X-CSRFToken`)

---
### 2. 🔒 Basic Authentication

Sends `username:password` base64-encoded in the header on every request.

```python
'DEFAULT_AUTHENTICATION_CLASSES': [
    'rest_framework.authentication.BasicAuthentication',
]
```

**✅ Pros**

- Extremely simple, no extra libraries
- Fine for quick internal tools/testing

**❌ Cons**

- Credentials sent on _every_ request (even if HTTPS-encrypted, risky)
- No expiry, no revocation
- 🚫 Never use in production public APIs

---
### 3. 🎟️ Token Authentication (DRF built-in)

`rest_framework.authtoken` — one static token per user, stored in DB.

```python
INSTALLED_APPS += ['rest_framework.authtoken']

'DEFAULT_AUTHENTICATION_CLASSES': [
    'rest_framework.authentication.TokenAuthentication',
]
```

**✅ Pros**

- Stateless-ish (no cookies), works well for mobile/SPA
- Simple mental model — one token, forever, until revoked

**❌ Cons**

- Token doesn't expire by default (security risk)
- One token per user (no multiple devices without extra work)
- DB lookup on every request → extra query overhead

---
### 4. 🎫 JWT Authentication (`djangorestframework-simplejwt`)

Self-contained signed token carrying claims (user id, expiry, etc).

**✅ Pros**

- Stateless — no DB hit to validate signature (only claims are decoded)
- Built-in expiry + refresh token flow
- Works great across mobile, SPA, microservices
- Can carry custom claims (roles, permissions) inside the token itself

**❌ Cons**

- Can't easily revoke a single token before expiry (needs blacklist app)
- Larger payload than a simple token (sent on every request)
- If secret key leaks → all tokens compromised
- Misconfiguration (e.g., storing in localStorage) → XSS risk

---
### 5. 🌐 OAuth2 (`django-oauth-toolkit`)

Delegated authorization — third parties get scoped access without your password.

**✅ Pros**

- Industry standard for "Login with X" / third-party API access
- Fine-grained scopes
- Supports multiple grant types (auth code, client credentials, PKCE)

**❌ Cons**

- Significantly more complex to set up and reason about
- Overkill if you just need "my app's users log into my app"

---
### 6. 🗝️ API Key Authentication

A static key issued per client/service (not per human user).

**✅ Pros**

- Simple for server-to-server / third-party integrations
- Easy to issue, rotate, and scope per client

**❌ Cons**

- Not meant for representing an individual end user
- If leaked, key must be manually rotated (no auto-expiry unless built yourself)

---
## ⚖️ Comparison Table

|Method|Stateless?|Expiry|Best for|Mobile-friendly|
|---|---|---|---|---|
|🍪 Session|❌ No|Session-based|Server-rendered / same-origin web apps|❌|
|🔒 Basic|✅ Yes|❌ None|Quick internal/testing only|⚠️|
|🎟️ Token (DRF)|⚠️ Semi (DB lookup)|❌ None (manual)|Simple SPA/mobile APIs|✅|
|🎫 JWT|✅ Yes|✅ Built-in|SPA, mobile, microservices|✅|
|🌐 OAuth2|✅ Yes|✅ Built-in|Third-party delegated access|✅|
|🗝️ API Key|✅ Yes|❌ Manual|Server-to-server|➖ N/A|

---
## 🎫 JWT Deep Dive

### 📦 Structure

A JWT has 3 base64url-encoded parts separated by dots:

```
header.payload.signature
```

- **Header** 🏷️ → algorithm + token type (`{"alg": "HS256", "typ": "JWT"}`)
- **Payload** 📄 → claims (`user_id`, `exp`, `iat`, custom claims)
- **Signature** ✍️ → HMAC/RSA signature to verify integrity

> [!warning] Not Encrypted JWT payload is only **encoded**, not encrypted. Never put secrets (passwords, card numbers) in the claims — anyone can base64-decode and read it.

### 🔄 Access + Refresh Token Flow

1. User logs in with credentials → server returns **access token** (short-lived, e.g. 5–15 min) + **refresh token** (long-lived, e.g. 7 days)
2. Client sends access token in `Authorization: Bearer <token>` header on each request
3. When access token expires → client calls `/token/refresh/` with refresh token → gets a new access token
4. When refresh token expires → user must log in again

```mermaid
sequenceDiagram
    Client->>Server: POST /login (credentials)
    Server-->>Client: access_token + refresh_token
    Client->>Server: GET /api/resource (Bearer access_token)
    Server-->>Client: 200 OK
    Note over Client,Server: ... time passes, access token expires ...
    Client->>Server: POST /token/refresh (refresh_token)
    Server-->>Client: new access_token
```

---
## 🛠️ Implementing JWT with SimpleJWT

### 1️⃣ Install

```bash
pip install djangorestframework-simplejwt
```
### 2️⃣ Settings

```python
# settings.py
INSTALLED_APPS = [
    ...
    'rest_framework',
    'rest_framework_simplejwt',
]

REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework_simplejwt.authentication.JWTAuthentication',
    ],
}

from datetime import timedelta

SIMPLE_JWT = {
    'ACCESS_TOKEN_LIFETIME': timedelta(minutes=15),
    'REFRESH_TOKEN_LIFETIME': timedelta(days=7),
    'ROTATE_REFRESH_TOKENS': True,       # 🔁 issue a new refresh token on refresh
    'BLACKLIST_AFTER_ROTATION': True,    # 🚫 invalidate old refresh tokens
    'ALGORITHM': 'HS256',
}
```
### 3️⃣ URLs

```python
# urls.py
from rest_framework_simplejwt.views import (
    TokenObtainPairView,
    TokenRefreshView,
    TokenBlacklistView,
)

urlpatterns = [
    path('api/token/', TokenObtainPairView.as_view(), name='token_obtain_pair'),
    path('api/token/refresh/', TokenRefreshView.as_view(), name='token_refresh'),
    path('api/token/blacklist/', TokenBlacklistView.as_view(), name='token_blacklist'),  # for logout
]
```

> [!tip] Blacklist app Add `'rest_framework_simplejwt.token_blacklist'` to `INSTALLED_APPS` and run migrations to enable logout/revocation via blacklisting.
### 4️⃣ Protecting a View

```python
from rest_framework.views import APIView
from rest_framework.permissions import IsAuthenticated
from rest_framework.response import Response

class ProfileView(APIView):
    permission_classes = [IsAuthenticated]

    def get(self, request):
        return Response({'username': request.user.username})
```
### 5️⃣ Custom Claims (e.g., role-based)

```python
from rest_framework_simplejwt.serializers import TokenObtainPairSerializer
from rest_framework_simplejwt.views import TokenObtainPairView

class MyTokenObtainPairSerializer(TokenObtainPairSerializer):
    @classmethod
    def get_token(cls, user):
        token = super().get_token(user)
        token['role'] = user.role          # 🏷️ custom claim
        token['is_staff'] = user.is_staff
        return token

class MyTokenObtainPairView(TokenObtainPairView):
    serializer_class = MyTokenObtainPairSerializer
```
### 6️⃣ Client Usage

```http
GET /api/profile/ HTTP/1.1
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

---

## 🛡️ Authorization (Permissions) in DRF

Authentication tells DRF _who_ `request.user` is. Permissions decide _what_ they can do.

```python
from rest_framework.permissions import BasePermission

class IsOwner(BasePermission):
    def has_object_permission(self, request, view, obj):
        return obj.owner == request.user
```

Common built-ins:

- `IsAuthenticated` ✅ — must be logged in
- `IsAdminUser` 👑 — staff only
- `AllowAny` 🌍 — public
- `IsAuthenticatedOrReadOnly` 👀 — read for all, write for logged-in users

**Combining JWT claims with permissions** — read custom claims off `request.auth` (the token) inside a permission class for role-based access without extra DB hits:

```python
class IsAdminRole(BasePermission):
    def has_permission(self, request, view):
        return request.auth.get('role') == 'admin'
```

---
## ⚠️ Common Pitfalls

- 🚨 Storing JWT in `localStorage` → vulnerable to XSS. Prefer `httpOnly` secure cookies for web clients when possible.
- 🚨 Long-lived access tokens with no blacklist → stolen token stays valid until it expires naturally.
- 🚨 Forgetting `ROTATE_REFRESH_TOKENS` + `BLACKLIST_AFTER_ROTATION` → refresh tokens can be reused indefinitely if leaked.
- 🚨 Putting sensitive data in the payload — it's readable by anyone, just not editable.
- 🚨 Mixing SessionAuthentication + JWT without disabling CSRF checks correctly for the JWT-only endpoints.

---
## ✅ Best Practices Checklist

- [ ] Access tokens short-lived (5–15 min)
- [ ] Refresh tokens rotated + blacklisted after use
- [ ] HTTPS enforced everywhere (JWTs are only signed, not encrypted)
- [ ] Sensitive data never placed in JWT payload
- [ ] Blacklist/logout endpoint implemented
- [ ] Role/permission claims validated server-side, not trusted blindly from client
- [ ] Rate limiting on `/token/` endpoints to prevent brute force