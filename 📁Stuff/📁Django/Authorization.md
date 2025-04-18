```table-of-contents
```
---
## 🔐 What is Authorization?

**Authorization** is the process of determining **what an authenticated user is allowed to do**.  
It answers the question:

> ✅ "Now that we know _who_ you are... what are you _allowed_ to do?"
---
## 💡 Quick Example

Imagine you're at a theme park:

- You show your **ticket** 🎟️ at the entrance—this is **authentication** (proving your identity).
- Then you’re told which rides you can go on 🎢, and what areas you can enter 🚫—this is **authorization**.
---
## 🧩 How It Works (Step-by-Step)

1. **Authentication First** 🧍🔑
    - You log in to the system. The system verifies who you are.
    - (E.g., username + password, Google login via [[OAuth]])
2. **Authorization Next** 🎯🔐
    - Based on your identity, the system checks **what you’re allowed to access or do**.
---
## 🛠️ Common Forms of Authorization

| Type                                      | Description                                                    |
| ----------------------------------------- | -------------------------------------------------------------- |
| **Role-Based Access Control (RBAC)**      | Access is based on user roles (e.g., Admin, Editor, Viewer)    |
| **Permission-Based Access**               | Access is based on specific actions (e.g., "can_edit_posts")   |
| **Attribute-Based Access Control (ABAC)** | Access based on user attributes, resource type, or environment |
| **Scope-Based ([[OAuth]])**               | In [[OAuth]], scopes define what an app is allowed to access   |

---
## 🔄 Authentication vs Authorization

|🔑 Authentication|🔐 Authorization|
|---|---|
|Who are you? 👤|What can you do? 🎯|
|Verified by login|Verified by policies/roles|
|Happens **first**|Happens **after** authentication|
|You get a token or session|You get permissions/access|

---
## 📦 In Django (Mini Preview)

Django supports **authorization** through:

- **Permissions** (`is_staff`, `can_add_post`, etc.) ✅
- **Groups** (bundles of permissions) 👥
- **Custom decorators** like `@login_required` and `@permission_required` 🛡️
- **DRF permissions** (`IsAuthenticated`, `IsAdminUser`, `CustomPermission`) 🔒

> Want a Django example after this? I can walk you through it step-by-step. 👨‍💻
---
## 🔐 Real-World Examples of Authorization

|Scenario|Auth Behavior|
|---|---|
|Admin editing a user|✅ Allowed (has `admin` role)|
|Viewer trying to delete data|❌ Denied (read-only access)|
|App requesting your Gmail access via OAuth|✅ Allowed only to read, not send emails (limited scope)|

---
## 🧠 Summary

- **Authorization** = Control over _what you can do_
- Happens _after_ **authentication**
- Usually implemented using **roles**, **permissions**, or **scopes**
- Keeps systems **secure** by enforcing boundaries 🔐
---
