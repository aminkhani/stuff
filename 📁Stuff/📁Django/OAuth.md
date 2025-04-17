## What is OAuth? 🤔

**OAuth** (Open Authorization) is a **secure framework** that allows third-party apps to access your data **without sharing your password**. Think of it as a "token" that lets someone in without giving them your house keys! 🏡🔑

---
## Why Do We Need OAuth? 🚪

- **Security 🔐**: OAuth lets apps access your data without needing your password. Instead, they use a **token** which is limited in what it can do.
- **Convenience 📱**: OAuth allows you to log in to apps using existing accounts like Google, Facebook, or Twitter—no need to remember another password!
- **Granular Permissions 🔑**: You can control what data a third-party app can access, like "view my calendar" but not "edit my contacts."
---
## How Does OAuth Work? 🤖

OAuth operates in these **roles** that work together:

1. **User (Resource Owner)** 👤: This is **you**, the person who owns the data.
2. **Client (App)** 📱: The third-party app or service (e.g., fitness app) that wants to access your data.
3. **Authorization Server** 🏦: The system that verifies your identity and gives the **access token**.
4. **Resource Server** 🗄️: The server where your data is stored (e.g., Google Drive, Facebook).
---
## OAuth Flow 💡

Here’s how OAuth works step-by-step:

1. **Request Authorization** 🔑:
	- The **client app** asks for permission to access your data.
    - You are redirected to the **Authorization Server** (e.g., Google, Facebook).
2. **Grant Permission ✅**:
    - You log in to the **Authorization Server** and approve what data the client app can access.
    - For example, “Allow this app to read my calendar.”
3. **Authorization Code 📜**:
    - If you approve, the **Authorization Server** sends an **authorization code** back to the client.
4. **Access Token 🎟️**:
    - The client app exchanges the **authorization code** for an **access token**, which it can use to access your data. This token is time-limited and can be scoped (only allowing certain actions).
5. **Access Data 📂**:
    - The client app uses the **access token** to access your data from the **Resource Server** (e.g., Google Drive, Facebook).
6. **Refresh Token 🔄**:
    - If the **access token** expires, the app can use a **refresh token** to get a new one without bothering you again.
---
## Real‑World Example 🏃‍♂️📅

Let’s say you want to use a **fitness tracking app** that wants to access your **Google Fit** data (steps, activity, etc.):

1. **Fitness App (Client)** 📱: The app asks for permission to access your **Google Fit** data.
2. **Google (Authorization Server)** 🏦: You log in to Google and approve the fitness app’s access to your data.
3. **Fitness App Receives Token 🎟️**: Google sends the app an **access token**.
4. **Fitness App Accesses Google Fit Data 📊**: The fitness app uses the token to access your step data, which you can then view in the app.
---
## Why is OAuth Useful? 🌍

- **Secure 🔒**: You don’t give your password to third-party apps, just a token that limits their access.
- **Flexible 🔑**: You can grant specific permissions—like allowing an app to read your calendar but not edit it.
- **Widely Used 🌐**: OAuth is used by companies like **Google**, **Facebook**, **Twitter**, and more to make your life easier.
---
## Key Terms 📖

- **Authorization Server** 🏦: The system that issues the **access token** (e.g., Google, Facebook).
- **Access Token** 🎟️: A limited-time key that gives a client app permission to access your data.
- **Refresh Token** 🔄: A long-lived token used to refresh the **access token** when it expires.
- **Scope** 📑: Defines the level of access the client app gets (e.g., read-only, write permissions).
---
### Summary 📝

- **OAuth** lets third-party apps access your data securely without sharing your password.
- It uses a **token-based system** with specific access permissions to keep things safe.
- OAuth helps make logging into apps easier and ensures that your data stays private! 🛡️

> Let’s dive into how OAuth 2.0 can handle **authorization** in Django—both as an **OAuth provider** (issuing tokens to clients) and as a **client** (consuming third‑party OAuth services). 🔐✨

## 🏗️ Acting as an OAuth 2.0 **Provider** with `django-oauth-toolkit`

### 1. Install & Configure

```bash
pip install django-oauth-toolkit
```

```python
# settings.py
INSTALLED_APPS += [
    'oauth2_provider',
]

MIDDLEWARE += [
    'oauth2_provider.middleware.OAuth2TokenMiddleware',
]

REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': (
        'oauth2_provider.contrib.rest_framework.OAuth2Authentication',
    ),
    'DEFAULT_PERMISSION_CLASSES': (
        'rest_framework.permissions.IsAuthenticated',
    ),
}
```

### 2. Migrate & Register

```bash
python manage.py migrate oauth2_provider
```
- Create an OAuth **Application** (client) via Django admin:
    - **Client type:** Confidential
    - **Authorization grant:** Authorization code
    - **Redirect URIs:** `https://yourapp.com/callback`
### 3. URL Routes
```python
# urls.py
from django.urls import path, include

urlpatterns = [
    # OAuth2 endpoints:
    path('o/', include('oauth2_provider.urls', namespace='oauth2_provider')),
    # Your API views:
    path('api/', include('your_app.api_urls')),
]
```
### 4. Protecting Views

```python
from oauth2_provider.decorators import protected_resource
from django.http import JsonResponse

@protected_resource(scopes=['read'])
def my_protected_view(request):
    return JsonResponse({'msg': 'Hello, authorized user!'})
```

- **Scopes** let you partition access (e.g., `read`, `write`, `admin`).
- Clients must request scopes when obtaining the code/token.

> 🚀 **Result:** Your Django app now issues and validates OAuth 2.0 tokens for any authorized client!
---
## 🔗 Acting as an OAuth 2.0 **Client** with `django-allauth` or `social-auth-app-django`

### 1. Install

```bash
pip install django-allauth # or pip install social-auth-app-django
```
### 2. Configure Providers

```python
# settings.py (for django-allauth)
INSTALLED_APPS += [
    'allauth', 'allauth.account', 'allauth.socialaccount',
    'allauth.socialaccount.providers.google',
]

AUTHENTICATION_BACKENDS = [
    'django.contrib.auth.backends.ModelBackend',
    'allauth.account.auth_backends.AuthenticationBackend',
]

SITE_ID = 1

SOCIALACCOUNT_PROVIDERS = {
    'google': {
        'SCOPE': ['email', 'profile'],
        'AUTH_PARAMS': {'access_type': 'online'},
    }
}
```
### 3. URL & Templates

```python
# urls.py
urlpatterns += [
    path('accounts/', include('allauth.urls')),
]
```

- Users click **“Log in with Google”** → redirected to Google’s consent page → upon success, `allauth` obtains an access token and creates or updates the local `User`.    
### 4. Accessing the Token

```python
# in a view
from allauth.socialaccount.models import SocialToken

def profile(request):
    token = SocialToken.objects.get(account__user=request.user, account__provider='google')
    # Use token.token to call Google APIs
```
> 🔒 **Use case:** Let users sign in via Google/Facebook/etc., then call their APIs (e.g., fetch Google Calendar).
---
## 🚨 Security & Best Practices

- **HTTPS only:** Always run OAuth flows over TLS. 🔐
- **Keep secrets safe:** Store client secrets in environment variables or a secrets manager. 🗝️
- **Use short-lived tokens & refresh tokens:** Limits exposure if stolen. ⏲️
- **Validate scopes & audiences:** Ensure incoming tokens have the right permissions. ✅
- **Regular audits:** Rotate secrets, revoke unused tokens, and monitor logs. 📊
---
### 🎯 Quick Recap

- **Provider (`django-oauth-toolkit`)** → Issue & manage access tokens for your APIs.
- **Client (`django-allauth`/`social-auth-app-django`)** → Let your users log in via third‑party OAuth services and use those tokens.
- Always enforce **HTTPS**, scope checks, and secret management.
---
