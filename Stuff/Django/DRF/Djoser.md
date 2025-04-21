## 📦 What Is Djoser?

Djoser is a third‑party Django app that provides a set of **RESTful** endpoints for user **authentication** and management:

- ✅ Registration
- ✅ Login & logout
- ✅ Password reset & set
- ✅ Token (JWT/DRF‑token) management

It “just works” with **DRF** serializers and views, so you don’t have to hand‑roll all of these common features.

---
## ⚙️ Installation & Setup

### 1. Install via pip
```bash
pip install djoser
```
### 2. Add to your `INSTALLED_APPS`
```python
INSTALLED_APPS = [
  # ...
  'rest_framework',
  'rest_framework.authtoken',  # if using DRF Token
  'djoser',
]
```
### 3. Configure authentication in `settings.py`
```python
REST_FRAMEWORK = {
  'DEFAULT_AUTHENTICATION_CLASSES': (
    'rest_framework.authentication.TokenAuthentication',
    # or for JWT: 'rest_framework_simplejwt.authentication.JWTAuthentication',
  ),
}

# (Optional) Djoser settings, e.g. to require email activation:
DJOSER = {
  'USER_CREATE_PASSWORD_RETYPE': True,
  'SEND_ACTIVATION_EMAIL': True,
  'PASSWORD_RESET_CONFIRM_URL': 'password/reset/confirm/{uid}/{token}',
  'ACTIVATION_URL': 'activate/{uid}/{token}',
  # etc.
}
```
### 4. Hook up the URLs
```python
from django.urls import path, include

urlpatterns = [
  # your app URLs…
  path('api/auth/', include('djoser.urls')),              # core endpoints
  path('api/auth/', include('djoser.urls.authtoken')),    # token endpoints
  # or for JWT:
  # path('api/auth/', include('djoser.urls.jwt')),
]
```
---
## 🔗 Key Endpoints & Examples

| Endpoint                                           | Action                     | Sample Request                                          |
| -------------------------------------------------- | -------------------------- | ------------------------------------------------------- |
| **POST** `/api/auth/users/`                        | Register new user          | `{ "email":"you@example.com", "password":"p@ss" }`      |
| **POST** `/api/auth/token/login/`                  | Obtain auth token          | `{ "email":"you@example.com", "password":"p@ss" }`      |
| **POST** `/api/auth/token/logout/`                 | Invalidate token           | _Header:_ `Authorization: Token <your_token>`           |
| **POST** `/api/auth/users/me/`                     | Get or update current user | _Header:_ `Authorization: Token <your_token>`           |
| **POST** `/api/auth/users/reset_password/`         | Send password reset email  | `{ "email":"you@example.com" }`                         |
| **POST** `/api/auth/users/reset_password_confirm/` | Confirm new password       | `{ "uid":"XYZ", "token":"ABC", "new_password":"new!" }` |

---
## 🛠️ Customizing Djoser

If you need custom behavior (e.g. extra fields on registration), you can override serializers:

```python
# myapp/serializers.py
from djoser.serializers import UserCreateSerializer
from .models import User

class MyUserCreateSerializer(UserCreateSerializer):
    class Meta(UserCreateSerializer.Meta):
        model = User
        fields = ('id','email','first_name','last_name','password')
```
and tell Djoser to use it:
```python
DJOSER = {
  'SERIALIZERS': {
    'user_create': 'myapp.serializers.MyUserCreateSerializer',
  }
}
```
---
## 🚀 Why Use Djoser?

- 🚧 **Saves development time**—no need to build auth flows from scratch.
- 🔄 **Fully DRF‑compatible**—plugs into your existing API.
- ⚙️ **Highly configurable**—override only what you need.
---
