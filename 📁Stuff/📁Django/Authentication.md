## 🔑 What is Authentication?

**Authentication** is the process of **verifying who someone is**.

It answers the question:

> **"Who are you?" 👤**  
> (And can you prove it?)

Without authentication, a system has no idea **who is making a request** or whether to even consider it.

---
## 🎢 Real-Life Analogy

Think of it like **checking in at the airport**:

- 🧍 You walk up to the gate agent
- 🪪 You show your **passport** or **ID**
- ✈️ If it matches your face & ticket, you’re allowed in

That **identity check** is **authentication**.

---
## 🧩 How Authentication Works

### 1. **User Identifies Themselves** 👤

You say “Hi, I’m Sarah!”

### 2. **User Provides Credentials** 🗝️

You give your:

- 🔡 Username or email
- 🔒 Password (or 2FA code, fingerprint, etc.)

### 3. **System Verifies You** ✅

The system checks:

- Do the credentials match what's stored?
- Is the account active?

If all’s good → you’re authenticated and logged in! 🎉

---
## 🧪 Types of Authentication

| Type                           | Description                         | Emojis |
| ------------------------------ | ----------------------------------- | ------ |
| **Password-Based**             | Most common; username + password    | 🔐💬   |
| **Token-Based (e.g. [[JWT]])** | Tokens passed with each request     | 🎟️📦  |
| **OAuth (3rd-party login)**    | Login via Google, Facebook, GitHub  | 🌐🔗   |
| **Multi-Factor (MFA)**         | Extra step: code, fingerprint, etc. | 📱🧬   |
| **Biometric**                  | Use face, fingerprint, iris, etc.   | 👁️🧠  |

---
## 🔄 Authentication vs Authorization

|Authentication 🔑|Authorization 🔐|
|---|---|
|Who are you? 👤|What can you do? 🎯|
|Happens first ⏱️|Happens after authentication|
|Verified by login|Verified by roles, permissions|
|You receive a token/session|You gain access to resources|

---
## 📦 Authentication in Django

Django makes authentication easy! 🛠️

- Built-in user model (`User`) 👤
- Login system with sessions 🍪
- Password hashing 🔒
- Support for OAuth, JWT, 2FA 🔌
### Examples:

```python
from django.contrib.auth import authenticate, login

user = authenticate(username='sarah', password='secret123')
if user:
    login(request, user)  # 🎉 You're authenticated!
```
Or with Django REST Framework:

```python
from rest_framework.permissions import IsAuthenticated

@api_view(['GET'])
@permission_classes([IsAuthenticated])
def my_secure_api(request):
    return Response({"message": "You're logged in! 👋"})
```
## 🛡️ Why is Authentication Important?

- 🔐 **Protects sensitive data**
- 👤 **Identifies users correctly**
- 🚫 **Blocks unauthorized access**
- 🔄 **Enables audit trails and user-specific features**
---
## 🔁 Common Authentication Mistakes

|Mistake|Why it's bad|Emoji|
|---|---|---|
|Storing passwords as plain text|Huge security risk if breached|⚠️😱|
|Weak passwords allowed|Easy to guess/brute-force|🔓🧠|
|No session timeout|Leaves accounts vulnerable|⏰🔓|

---
## ✅ Summary

- **Authentication** = Proving **who you are** 👤
- It’s step **one** before authorization 🛂
- It can be done with passwords, tokens, OAuth, or biometrics
- Django and DRF offer **powerful built-in tools** to manage authentication securely 💪
---
