## 🗺️ 1. The Big Picture — Who Handles What

```
 "Who are you?"                        →  🪪 Authentication (IdP, MFA)
 "Prove it's really you, more than
  once, in different ways"             →  🔐 MFA
 "What are you allowed to do?"         →  🎚️ Authorization (RBAC/ABAC)
 "You're an admin — extra scrutiny"    →  👑 PAM
 "You're connecting from outside —
  don't just VPN into everything"      →  🧩 ZTNA
```

> [!success] One-line memory hook **Authentication proves who you are. Authorization decides what you can touch. Everything else in this note is about making both of those harder to fake or abuse.**

---
## 🪪 2. IdP — Identity Provider

> [!tip] TL;DR The **central service that authenticates users** and issues proof of identity to other applications, so you don't need a separate password for every single app.

> [!warning] Don't confuse with IDP/IDS This is the **Identity** Provider (SSO context). It's a totally different thing from the network-security **IDP** (Intrusion Detection/Prevention) in [[Endpoint-Security-Components]] — same three letters, unrelated concepts.

**How it fits into Single Sign-On (SSO):**

```
 User → tries to log into App
        ↓
 App redirects user to the IdP (e.g., Okta, Azure AD/Entra ID, Google Workspace)
        ↓
 User authenticates ONCE at the IdP (password + MFA)
        ↓
 IdP issues a signed token back to the App
        ↓
 App trusts the token → user is logged in, no separate password needed
```

**The protocols that carry this handshake:**

| Protocol                  | Emoji | Common use case                                                       |
| ------------------------- | ----- | --------------------------------------------------------------------- |
| **SAML**                  | 📜    | Older, XML-based, common in enterprise SSO                            |
| **OAuth 2.0**             | 🔑    | Authorization — "let App B access some of my data in App A"           |
| **OIDC** (OpenID Connect) | 🪪    | Authentication layer built _on top of_ OAuth 2.0 — "who is this user" |

> [!note] OAuth vs OIDC in one line OAuth 2.0 answers **"what can this app access?"** OIDC answers **"who is this user?"** — OIDC is literally OAuth 2.0 plus an identity layer on top.

---
## 🔐 3. MFA — Multi-Factor Authentication

> [!tip] TL;DR Requiring **more than one type of proof** before trusting a login — because a stolen password alone shouldn't be enough.

**The three classic factor categories:**

|Factor type|Emoji|Examples|
|---|---|---|
|**Something you know**|🧠|Password, PIN|
|**Something you have**|📱|Authenticator app code, hardware key (YubiKey), SMS code|
|**Something you are**|👆|Fingerprint, face ID|

**Not all MFA is equally strong:**

|Method|Strength|Why|
|---|---|---|
|SMS codes|⚠️ Weakest|Vulnerable to SIM-swapping attacks|
|Authenticator app (TOTP)|✅ Good|Codes generated locally, not sent over a network|
|Push notification approval|✅ Good|But vulnerable to "MFA fatigue" attacks (spamming approval requests until one is accidentally approved)|
|Hardware security key (FIDO2/WebAuthn)|🏆 Strongest|Phishing-resistant — cryptographically tied to the real site, can't be tricked by a fake login page|

> [!warning] MFA fatigue / prompt bombing A real attack pattern: attacker already has the password, then spams push-approval requests until the tired/annoyed user taps "Approve" just to make it stop. This is why phishing-resistant MFA (hardware keys) is increasingly recommended over push notifications for high-privilege accounts.

---
## 🎚️ 4. Authorization Models — RBAC vs ABAC

> [!tip] TL;DR Once you know _who_ someone is, authorization decides _what they're allowed to do_.

|Model|Emoji|How it works|Example|
|---|---|---|---|
|**RBAC** (Role-Based Access Control)|🎭|Access tied to a **role** the user is assigned|"Editor" role can edit posts, "Viewer" role can only read|
|**ABAC** (Attribute-Based Access Control)|🧩|Access decided by **attributes/context** at request time|"Allow if department = Finance AND time = business hours AND device is managed"|

> [!note] RBAC vs ABAC trade-off RBAC is simpler to reason about and audit but gets unwieldy as roles multiply ("role explosion"). ABAC is more flexible and fine-grained but harder to audit because the rules are more like code than a simple list.

**Related principle worth remembering:** ⚖️ **Principle of Least Privilege** — always grant the _minimum_ access needed to do the job, nothing more. Almost every access-control model above exists to help enforce this in practice.

---
## 👑 5. PAM — Privileged Access Management

> [!tip] TL;DR Extra layers of control specifically around **admin/root/superuser accounts**, because compromising one of these is far more damaging than a regular user account.

**Core PAM techniques:**

- 🗄️ **Credential vaulting** — admin passwords aren't memorized/shared; they're checked out from a vault when needed
- ⏱️ **Just-in-Time (JIT) access** — privileged access is granted temporarily for a specific task, then automatically revoked
- 📹 **Session recording** — privileged sessions can be recorded/logged for audit purposes
- 🚫 **No standing privilege** — the ideal end state: nobody has permanent admin rights sitting there waiting to be stolen; access is elevated only when actually needed

**Why this matters more than regular account security:** a compromised regular user account might expose their own data. A compromised admin account can expose _everything_ — this is why PAM gets its own dedicated tooling and process, separate from normal IAM.

---
## 🧩 6. ZTNA — Zero Trust Network Access

> [!tip] TL;DR The modern replacement for "connect to VPN → now you're inside the network with broad access." ZTNA grants access **per-application**, continuously verified, instead of per-network.

**VPN vs ZTNA:**

| Traditional VPN             | ZTNA                                                        |                                                                  |
| --------------------------- | ----------------------------------------------------------- | ---------------------------------------------------------------- |
| Access granularity          | Whole network segment                                       | One specific application at a time                               |
| Trust model                 | Trusted once connected                                      | Continuously re-verified                                         |
| Blast radius if compromised | Large — attacker can reach anything on that network segment | Small — attacker only reached the one app they had a session for |
| Fits Zero Trust principles? | ❌ Not really (implicit trust once inside)                   | ✅ Yes, by design                                                 |

**Where this connects to your DevSecOps work:** ZTNA is often how remote engineers securely reach internal admin panels, staging environments, or internal APIs — instead of a flat VPN that would technically expose every internal service to anyone who's connected.

---
## 🧩 7. Where This Fits in NIST CSF (tying back to [[Security-Frameworks]])

| Control                                                 | NIST CSF Function                                                            |
| ------------------------------------------------------- | ---------------------------------------------------------------------------- |
| MFA, RBAC/ABAC enforcement, PAM vaulting                | 🛡️ **Protect**                                                              |
| Unusual login detection (impossible travel, new device) | 👀 **Detect** — feeds into [[Detection-and-Response]] (XDR identity signals) |
| Auto-revoking a compromised session/token               | 🚨 **Respond**                                                               |
| Identity/asset inventory (who has access to what)       | 🔎 **Identify**                                                              |

---
## 🧠 Quick Review (Self-Quiz)

> [!question]- What's the difference between authentication and authorization, in one sentence each? Authentication proves _who you are_. Authorization decides _what you're allowed to do_ once you're known.

> [!question]- What's the difference between OAuth 2.0 and OIDC? OAuth 2.0 handles _authorization_ — what an app can access. OIDC adds an _authentication_ (identity) layer on top of OAuth 2.0 — who the user actually is.

> [!question]- Why is SMS-based MFA considered weaker than an authenticator app or hardware key? SMS is vulnerable to SIM-swapping attacks, where an attacker convinces a carrier to port the victim's number to a new SIM they control.

> [!question]- What is an "MFA fatigue" attack? An attacker who already has the password spams push-notification approval requests repeatedly until the user accidentally (or out of annoyance) approves one.

> [!question]- What's the key difference between RBAC and ABAC? RBAC grants access based on a fixed role assignment. ABAC grants access dynamically based on attributes/context at request time (department, time of day, device state, etc.).

> [!question]- What does "no standing privilege" mean in PAM, and why is it the ideal end state? It means nobody holds permanent admin rights sitting idle — privileged access is only elevated temporarily (JIT) when actually needed, minimizing the window an attacker could exploit a stolen privileged credential.

> [!question]- How does ZTNA reduce blast radius compared to a traditional VPN? ZTNA grants access to one specific application at a time with continuous verification, so a compromised session only exposes that one app — not the entire network segment a VPN connection would expose.

---

🔗 Related: [[Endpoint-Security-Components]] · [[Detection-and-Response]] · [[Security-Frameworks]] · [[Cloud-CICD-Pipeline-Security]]