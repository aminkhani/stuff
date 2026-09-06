## 🗺️ 1. The Big Picture — Where Each Control Sits in the Pipeline

```
 💻 CODE           📦 BUILD            🚀 DEPLOY            ☁️ RUNTIME
 ─────────         ──────────          ───────────          ────────────
 SAST 🕳️           SCA 🔗              Secrets Mgmt 🔏      CSPM 🏗️
 Secrets scan 🔍    SBOM 📋            IaC scanning 🧱      DAST 🌐
                    Supply-chain ⛓️                          Runtime protection 👀
```

> [!success] One-line memory hook **"Shift Left, Watch Right"** — catch issues as early as possible in the pipeline (SAST/SCA on code & dependencies), but still watch the running system afterward (DAST/CSPM/runtime), because nothing catches everything.

---

## 🕳️ 2. SAST — Static Application Security Testing

> [!tip] TL;DR Scans your **source code, without running it**, looking for insecure patterns.

- 📖 Analyzes code structure/syntax directly (like a very security-focused linter)
- 🐛 Finds things like: SQL injection patterns, hardcoded credentials, unsafe deserialization, missing input validation
- ⚡ Runs fast, early — typically on every commit/PR in CI
- ❌ **Limitation:** doesn't understand runtime context, so it can flag things that aren't actually exploitable (false positives) and misses issues that only appear when the app is actually running

**Where it fits for you:** a SAST tool scanning your Django codebase would flag things like raw SQL string concatenation, unsafe `eval()`/`pickle` usage, or missing CSRF protections — before the code ever merges.

---

## 🌐 3. DAST — Dynamic Application Security Testing

> [!tip] TL;DR Tests the **running application from the outside**, like an automated attacker probing a live web app.

- 🎯 Sends real HTTP requests to your running app/API and observes the responses
- 🐛 Finds things like: actual exploitable XSS, broken auth, injection that SAST couldn't confirm, misconfigured headers
- 🐌 Slower than SAST (needs a running environment), usually run in staging, not on every commit
- ✅ **Strength:** finds _real, confirmed_ runtime vulnerabilities — fewer false positives than SAST, but only what it can reach through the app's interface

**SAST vs DAST at a glance:**

|SAST|DAST|
|---|---|---|
|Looks at|Source code (static)|Running app (dynamic)|
|Needs the app running?|❌ No|✅ Yes|
|Speed|Fast, per-commit|Slower, per-build/staging|
|False positives|Higher|Lower|
|Finds|Code-level patterns|Confirmed exploitable behavior|

> [!note] IAST/RASP (bonus terms you may see) **IAST** (Interactive AST) instruments the app _while it runs_ under normal testing, blending SAST/DAST strengths. **RASP** (Runtime Application Self-Protection) goes further and actively blocks attacks _in production_ from inside the running app.

---

## 🔗 4. SCA — Software Composition Analysis

> [!tip] TL;DR Scans your **dependencies** (npm/pip/etc. packages) for known vulnerabilities (CVEs) and license issues — because most of a modern app's code is actually _someone else's_ code.

- 📦 Builds an inventory of every direct **and transitive** dependency
- 🚨 Cross-references each version against vulnerability databases (e.g., the National Vulnerability Database)
- ⚖️ Also flags risky open-source **licenses** (e.g., copyleft licenses that could create legal obligations)
- 🔄 Needs to run continuously — a dependency that's safe today can get a new CVE tomorrow

**Where it fits for you:** scanning `requirements.txt`/`package.json` for a Django/Celery/DRF stack — e.g., catching a known-vulnerable version of a PostGIS-adjacent library before it ships.

---

## 🔏 5. Secrets Management

> [!tip] TL;DR Making sure API keys, DB passwords, and tokens are **never hardcoded or committed** — and are instead pulled dynamically from a secure store at runtime.

**Common failure mode this prevents:** a secret accidentally committed to Git — even if deleted in a later commit, it's still in Git history forever unless the history is rewritten.

**Key building blocks:**

- 🗝️ **Secrets vaults** (e.g., HashiCorp Vault, cloud KMS/Secrets Manager) — centralized, access-controlled secret storage
- 🔄 **Dynamic/short-lived secrets** — issued on-demand and auto-expiring, instead of one static password living forever
- 🔍 **Secrets scanning in CI** — a pre-commit or pipeline step that blocks a push if it detects a pattern that looks like a key/token
- 🚫 **Never in IaC files directly** — Terraform/CloudFormation variables should reference the vault, not contain plaintext secrets

---

## 🧱 6. IaC Security Scanning

> [!tip] TL;DR Scanning your **Terraform/CloudFormation/Kubernetes manifests** for misconfigurations _before_ they're ever applied to real infrastructure.

**Common things it catches:**

- 🪣 A storage bucket left publicly readable
- 🔓 A security group open to `0.0.0.0/0` on a sensitive port
- 🔑 Missing encryption-at-rest settings
- 👤 Overly broad IAM roles/policies (violates least-privilege)

**Why "shift left" matters here specifically:** fixing a misconfigured Terraform file in a PR review costs minutes. Finding the same misconfiguration after it's already live in production (via CSPM, below) costs an incident.

---

## 🏗️ 7. CSPM — Cloud Security Posture Management

> [!tip] TL;DR Continuously scans your **already-running cloud environment** for misconfigurations and drift — the runtime counterpart to IaC scanning.

- 👀 Watches live cloud accounts (AWS/Azure/GCP/Cloudflare) for risky configuration
- 📐 Compares actual state against a compliance baseline (CIS Benchmarks, SOC 2 controls, ISO 27001 controls)
- ⚠️ Catches **configuration drift** — someone manually changing something in the console that IaC scanning never saw, because it bypassed the pipeline entirely
- 🔗 Often feeds alerts into the same SIEM/XDR pipeline from [[Detection-and-Response]]

**IaC scanning vs CSPM in one line:** IaC scanning checks the _blueprint_ before you build; CSPM checks the _actual building_ after it's built (and keeps checking, forever).

---

## 📦 8. Supply-Chain Security — SBOM & SLSA

> [!tip] TL;DR Verifying not just _your_ code, but the **entire chain of tools, dependencies, and build steps** that produced your final shipped artifact.

- 📋 **SBOM** (Software Bill of Materials) — a complete, machine-readable inventory of everything inside a build artifact (direct + transitive dependencies, versions, licenses) — think "ingredient label" for software
- 🪜 **SLSA** (Supply-chain Levels for Software Artifacts) — a maturity framework (Levels 1–4) for how _trustworthy_ your build process itself is: is the build script tamper-proof? Are build steps reproducible? Is provenance cryptographically signed?
- 🎯 **Why this became a big deal:** high-profile attacks (e.g., SolarWinds-style incidents) came from compromising the _build pipeline itself_, not the final code — so trusting "the code looks fine" isn't enough anymore; you also need to trust _how it was built_

> [!warning] The core supply-chain question It's not just "is this dependency safe?" (SCA answers that). It's "**can I prove exactly what went into this artifact, and that nothing tampered with the build process itself?**" — that's what SBOM + SLSA are built to answer.

---

## 🧩 9. Where This Fits in NIST CSF (tying back to [[Security-Frameworks]])

|Control|NIST CSF Function|
|---|---|
|SAST / SCA / IaC scanning (pre-deploy)|🛡️ **Protect**|
|DAST / CSPM (continuous, post-deploy)|👀 **Detect**|
|Secrets rotation after a leak, incident triage|🚨 **Respond**|
|SBOM / dependency inventory|🔎 **Identify**|

---

## 🧠 Quick Review (Self-Quiz)

> [!question]- What's the fundamental difference between SAST and DAST? SAST scans source code statically, without running the app — fast but more false positives. DAST tests the running application dynamically, from the outside — slower but confirms real exploitability.

> [!question]- What does SCA scan for, and why does it need to run continuously rather than once? SCA scans dependencies (including transitive ones) for known CVEs and risky licenses. It must run continuously because a dependency that's safe today can have a new CVE disclosed tomorrow.

> [!question]- Why is committing a secret to Git dangerous even if you delete it in a later commit? Git history retains the old commit, so the secret is still recoverable from history unless it's explicitly rewritten/purged — and the secret should be rotated regardless.

> [!question]- What's the difference between IaC scanning and CSPM? IaC scanning checks your Terraform/CloudFormation _blueprint_ before deployment. CSPM continuously monitors the _already-running_ cloud environment, catching drift that bypassed the pipeline entirely.

> [!question]- What problem do SBOM and SLSA solve that SCA alone doesn't? SCA tells you if a known dependency is vulnerable. SBOM/SLSA go further — proving exactly what's inside a build artifact and that the build _process itself_ wasn't tampered with, which matters for supply-chain attacks that compromise the pipeline rather than the code.

> [!question]- Under NIST CSF, which function would DAST and CSPM fall under, versus SAST and SCA? DAST/CSPM (continuous, post-deploy checks) → **Detect**. SAST/SCA (pre-deploy checks) → **Protect**.

---

🔗 Related: [[Endpoint-Security-Components]] · [[Detection-and-Response]] · [[Security-Frameworks]]