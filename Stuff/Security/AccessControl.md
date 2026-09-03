```table-of-contents
title: **Table of Contents**
style: nestedList # TOC style (nestedList|nestedOrderedList|inlineFirstLevel)
minLevel: 0 # Include headings from the specified level
maxLevel: 0 # Include headings up to the specified level
includeLinks: true # Make headings clickable
hideWhenEmpty: false # Hide TOC if no headings are found
debugInConsole: false # Print debug info in Obsidian console
```
---
## 🧩 Authentication vs Authorization

**Authentication** (AuthN) answers *who are you* and yields a verified identity — password, token, client certificate, SSO assertion. **Authorization** (AuthZ) answers *what may this identity do to this specific resource, right now*, and yields an allow/deny verdict. AuthN happens once per session; AuthZ happens **on every request, for every object touched**. Most real breaches are not broken AuthN — they are a missing AuthZ check on one endpoint. See [Authentication](../Django/Authentication.md) and [Authorization](../Django/Authorization.md).

**Access control** is the machinery that turns "who" into "may". Like a hotel 🏨: reception verifies your passport (AuthN), while the keycard system decides which doors open, on which floor, until checkout (AuthZ).

---
## ⚙️ The policy machinery — PEP, PDP, PIP, PAP

Every access-control system has these four roles, even a single `if` statement. Naming them is what later lets you **externalize** policy instead of scattering it through views.

```text
  request -> [ PEP ] --"may alice DELETE /orders/42 ?"--> [ PDP ] -> allow | deny
              enforce            asks [ PIP ] for attributes (role, owner, dept)
                                 evaluates policy authored at [ PAP ] (admin / git)
```

| Component                             | Role                                         | In plain Django                              | In a policy-as-code stack             |
| ------------------------------------- | -------------------------------------------- | -------------------------------------------- | ------------------------------------- |
| **PEP** — Policy *Enforcement* Point  | Intercepts the request, applies the verdict   | DRF permission class, middleware, decorator   | Envoy/Nginx `ext_authz`, service SDK  |
| **PDP** — Policy *Decision* Point     | Evaluates policy, returns allow/deny          | `user.has_perm()` against the DB              | OPA sidecar evaluating Rego           |
| **PIP** — Policy *Information* Point  | Supplies the attributes policy needs          | `User`, `Group`, the object row itself        | LDAP/HR system, JWT claims, DB lookup |
| **PAP** — Policy *Administration* Point | Where policy is authored and versioned      | Django admin + migrations                     | Git repo of `.rego` files + CI        |

The consequence that matters: **the PEP must be unavoidable.** One view that forgets its permission class is one missing PEP, and no PDP anywhere will save you.

---
## 🧱 The models — DAC, MAC, RBAC, ABAC, PBAC, ReBAC

### 👤 DAC — Discretionary Access Control

The **owner of the object decides**. Permissions live on the object and are handed out at the owner's discretion: Linux file modes and ACLs, "share with anyone who has the link", S3 object ACLs, `GRANT` in PostgreSQL. Maximally flexible, and exactly how data leaks — one user re-shares a folder and nothing centrally notices.

### 🏷️ MAC — Mandatory Access Control

A **system-wide policy** labels subjects and objects, and **the user cannot delegate access even to files they own**. Military classification (Unclassified → Secret → Top Secret, plus compartments), SELinux/AppArmor types. This is the strongest containment there is: a compromised Nginx worker labelled `httpd_t` cannot read `/etc/shadow` *even as root*, because the kernel refuses.

> [!NOTE]
> The repo index lists **"MAD"** — that is a slip. The standard term is **MAC (Mandatory Access Control)**.

### 👥 RBAC — Role-Based Access Control

Users get **roles**, roles carry **permissions**; per user you manage only membership. Django groups + permissions, Kubernetes `Role`/`RoleBinding`, essentially every SaaS admin panel. Simple, auditable, the industry default — right up until the requirement becomes "but only their *own* articles, during business hours", which a role cannot express.

### 🧬 ABAC — Attribute-Based Access Control

The verdict is **computed from attributes** of subject, resource, action and environment: department, clearance, device posture, time of day, resource owner, IP range. "A nurse may read a chart if she is on that patient's care team and her shift is active." Expressive enough for real business rules; harder to audit, because "who can read this record?" becomes a query rather than a lookup.

### 📜 PBAC — Policy-Based Access Control / policy-as-code

ABAC's operational form: policy is **lifted out of the application** into a versioned, testable artifact evaluated by a dedicated engine — [OPA](../DevOps/OPA/README.md) with Rego, AWS IAM policy documents, Cedar. One policy serves the API, the CI pipeline and the Kubernetes admission controller, and changing a rule is a reviewed commit instead of redeploying six services.

### 🔗 ReBAC — Relationship-Based Access Control

Authorization follows the **graph of relationships** between subjects and objects, Google Zanzibar style (OpenFGA, SpiceDB, Ory Keto). The question is literally *"is `user:amin` in the `editors` relation of `doc:readme` — directly, through a group, through the parent folder, or by ownership?"* This is how Drive/GitHub-style nested sharing works, and it is the only model that scales to per-object **inherited** permissions without an N×M permission table.

---
## 🧪 Examples — DAC, RBAC → object-level, and a Rego policy

**DAC on Linux** — the owner grants, the kernel enforces:

```bash
chmod 640 report.csv                 # owner rw, group r, world nothing
chown amin:analysts report.csv
setfacl -m u:auditor:r report.csv    # POSIX ACL: one extra user, read-only
getfacl report.csv                   # -> user::rw-  user:auditor:r--  group::r--
```

**RBAC → object-level in DRF.** Django hands you model-level permissions; the per-object check is yours to write:

```python
from rest_framework.permissions import BasePermission, DjangoModelPermissions

class IsOwner(BasePermission):
    def has_object_permission(self, request, view, obj):
        return obj.owner_id == request.user.id       # object-level = YOUR job

class OrderViewSet(ModelViewSet):
    serializer_class = OrderSerializer
    permission_classes = [DjangoModelPermissions, IsOwner]   # role AND ownership

    def get_queryset(self):
        # Defence in depth: a foreign pk must 404, and a list endpoint must never
        # leak rows the object check would have denied one by one.
        return Order.objects.filter(owner=self.request.user)
```

**ABAC / PBAC in Rego** — attributes plus context, evaluated outside the application:

```rego
package orders.authz
import rego.v1

default allow := false                     # deny-by-default, always

allow if {                                 # the owner may always read
	input.action == "read"
	input.resource.owner == input.subject.id
}
allow if {                                 # support: in-region, during shift only
	input.action == "read"
	"support" in input.subject.roles
	input.resource.region == input.subject.region
	input.context.hour >= 8
	input.context.hour < 20
}
```

---
## ⚖️ DAC vs MAC vs RBAC vs ABAC vs PBAC vs ReBAC

| Model      | Policy expressed as        | Who can grant                | Scales with            | Auditability                          | Typical implementation                        | Pick it when                                          |
| ---------- | -------------------------- | ---------------------------- | ---------------------- | ------------------------------------- | --------------------------------------------- | ----------------------------------------------------- |
| **DAC**    | per-object ACL             | the object **owner**         | number of objects      | poor — grants are scattered            | Linux modes/ACLs, S3 ACLs, SQL `GRANT`        | Small trusted teams, personal file sharing            |
| **MAC**    | system-wide labels         | **nobody** except the admin  | a fixed label lattice  | excellent, and rigid                   | SELinux, AppArmor, classification levels      | Containing a compromised process; classified data     |
| **RBAC**   | role → permission set      | the role administrator       | number of **roles**    | good, until role explosion             | Django groups, Kubernetes RBAC                | Coarse, stable job functions. The default.            |
| **ABAC**   | rules over attributes      | the policy author            | number of **attributes** | medium — "who can read X?" is a query | XACML, IAM conditions, hand-written checks    | Context matters: time, region, device, ownership      |
| **PBAC**   | versioned policy code      | policy repo + CI reviewers   | number of services     | very good — git history + tests         | [OPA](../DevOps/OPA/README.md)/Rego, Cedar    | Many services must share one audited policy           |
| **ReBAC**  | relationship tuples (graph)| owners, transitively          | number of **relations** | good, if the tuple store is queryable  | Zanzibar, OpenFGA, SpiceDB                    | Nested sharing, per-object inherited permissions      |

Real systems are **hybrids**: RBAC for coarse roles, ownership/ReBAC per object, PBAC to keep rules out of application code. Start with RBAC and add object-level checks the moment two tenants share a table.

---
## 🚨 Where it goes wrong — and when NOT to use each model

- ❌ **Missing object-level checks (IDOR / BOLA).** `GET /api/orders/42/` returns someone else's order because the view checked the *role* and never the *object*. This is **#1 in the OWASP API Security Top 10**, it returns a clean `200 OK`, and no scanner flags it. Filter the queryset **and** check the object.
- ❌ **Role explosion.** `editor_eu_readonly_q3_finance` is RBAC being asked to express attributes. When role count approaches user count, you needed ABAC/ReBAC two quarters ago.
- ❌ **Privilege creep.** People change teams and keep the old groups forever. The fix is periodic **access reviews**, not good intentions.
- ❌ **Allow-by-default.** A view with no permission class, an OPA `default allow := true`, a DRF project left on `AllowAny` — all fail open. Deny explicitly at every layer.
- ❌ **DAC for regulated data** — you cannot prove who has access when any owner may re-share.
- ❌ **MAC as your business logic** — SELinux is superb at process containment and hopeless as an application authorization model; users cannot delegate anything.
- ❌ **ABAC/PBAC too early.** An OPA sidecar and a Rego bundle for a 3-role CRUD app is infrastructure you now have to operate. Django groups are enough until they genuinely aren't.
- ❌ **ReBAC without a tuple store** — recursive graph checks through your ORM on every request will die on latency. Use OpenFGA/SpiceDB or don't do ReBAC. And ❌ **enforcing in the UI only**: hiding the delete button is not access control, the endpoint is still there.

---
## 🔐 Security notes & production hardening

- **Deny by default, allow explicitly** — at every layer: DRF defaults, OPA `default allow := false`, database `GRANT`s, Kubernetes RBAC, firewall rules.
- **Least privilege per identity.** The Django DB role needs `SELECT/INSERT/UPDATE/DELETE` on its own schema, not `SUPERUSER`. Same discipline for CI tokens and cloud roles.
- **Separation of duties.** Whoever authors a policy should not be its only approver, and whoever deploys should not be its only reviewer — that is what stops one compromised account from being sufficient.
- **Break-glass accounts:** exactly one, MFA, sealed, alarmed on use, rotated afterwards — never a shared `admin/admin` "just in case". Keep the credential in [Vault](../DevOps/Vault/README.md) with an audit device enabled.
- **Access reviews** — quarterly for humans, immediately on offboarding, automated for service accounts. This is also precisely the evidence [ISO 27001](./ISO27001.md) asks you to produce.
- **Log the decision, not just the request:** subject, action, resource, verdict, policy version. Otherwise you cannot answer "who *could* have read this record in March?". And **never trust client-supplied identity** — `X-User-Id`, a `role` field in the body, an unverified JWT are all attacker-controlled; derive identity only from the verified [TLS](./TransportSecurity.md)/session/token context.

---
## 🐍 Django / Backend tie-in

Django's built-in system is **RBAC-ish and model-level**: `Group` → `Permission` → `add/change/delete/view_<model>`. It answers "may this user change *articles*", never "may this user change *this* article". **Object-level authorization is entirely on you.**

| Need                        | Tool                                                                              |
| --------------------------- | --------------------------------------------------------------------------------- |
| Per-object checks in DRF    | `has_object_permission` on a `BasePermission` — detail routes only                  |
| Per-object rows in the DB   | `django-guardian` — real per-object permissions stored in tables                    |
| Rule/predicate logic        | `django-rules` — permissions as small, testable Python predicates                  |
| Externalized policy         | [OPA](../DevOps/OPA/README.md) sidecar, called from one permission class            |

```python
REST_FRAMEWORK = {
    # Deny-by-default: a view that forgets its permission class still needs a login.
    "DEFAULT_PERMISSION_CLASSES": ["rest_framework.permissions.IsAuthenticated"],
}
```

> [!WARNING]
> `has_object_permission` is **not called** on `list` actions, nor for objects you fetch yourself. Always re-filter in `get_queryset()` with `filter(owner=self.request.user)`, and re-check ownership inside every custom `@action`. Multi-tenant leaks are almost always a queryset that forgot the tenant, not a missing permission class.

---
## 🧠 Summary

| Concept                 | Takeaway                                                                              |
| ----------------------- | ------------------------------------------------------------------------------------- |
| **AuthN vs AuthZ**      | Who you are (once) vs what you may touch (every request, every object)                 |
| **PEP/PDP/PIP/PAP**     | Enforce / decide / supply attributes / author — and the PEP must be unavoidable         |
| **Model ladder**        | DAC → MAC → RBAC → ABAC → PBAC → ReBAC: expressiveness up, auditability traded away     |
| **Django reality**      | Model-level RBAC for free; object-level checks and queryset filtering are yours          |
| **#1 failure**          | Missing object-level authorization (IDOR/BOLA) — a valid `200 OK` that no tool reports   |




