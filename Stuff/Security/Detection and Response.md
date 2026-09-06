## 🗺️ 1. The Big Picture — How the Pieces Connect

```
 📡 SIGNALS (endpoints, network, cloud, identity)
        ↓
 🕵️ EDR  →  🔗 XDR   (correlate signals into real threats)
        ↓
 📊 SIEM   (centralize logs + alerts from EVERYTHING, not just endpoints)
        ↓
 🤖 SOAR   (automate the repetitive response steps)
        ↓
 🏢 SOC    (the humans who watch it all and make judgment calls)
```

> [!success] One-line memory hook **EDR/XDR generate the smart alerts → SIEM stores & correlates ALL alerts → SOAR automates the boring parts of response → SOC is the team steering the whole thing.**

---

## 🕵️ 2. EDR — Endpoint Detection & Response

> [!tip] TL;DR Continuous recording + analysis of **everything happening on an endpoint**, so you can detect, investigate, and respond to threats that slipped past preventive controls (AV/HIPS/AMSI).

**What EDR actually does:**

- 📼 **Records telemetry continuously** — process creation, network connections, file writes, registry changes (like a "flight recorder" for the machine)
- 🔍 **Detects suspicious behavior chains**, not just single bad events (e.g., Word → PowerShell → outbound connection = suspicious chain even if each step looks fine alone)
- 🧵 **Lets analysts "thread pull"** — click one alert and see the full attack timeline before and after it
- 🛑 **Response actions**: isolate the host from the network, kill a process, quarantine a file, roll back changes — remotely, from a console

**EDR vs traditional AV/HIPS:**

||Antivirus / HIPS|EDR|
|---|---|---|
|Focus|Block known-bad **in the moment**|Investigate & respond **over time**|
|Data retention|Minimal|Weeks/months of telemetry|
|Best for|Prevention|Detection + investigation + response|
|Analyst usable?|Barely|Yes — built for hunting|

> [!warning] Common misconception EDR does **not replace** antivirus/HIPS — it's a different layer. Most real products (Defender, CrowdStrike, SentinelOne) bundle both prevention (AV/HIPS/AMSI-style) and EDR telemetry/response in one agent.

---

## 🔗 3. XDR — Extended Detection & Response

> [!tip] TL;DR Takes EDR's idea (correlate behavior over time) and **extends it beyond the endpoint** — network, cloud workloads, email, and identity signals all feed into one correlation engine.

**Why XDR exists:**

- 🧩 A real attack rarely stays on one endpoint — it touches network (lateral movement), identity (stolen credentials), cloud (privilege escalation), email (initial phishing)
- 🕳️ Looking at each source in isolation = you only see fragments
- 🔦 XDR correlates fragments from **all sources** into a single incident story

**EDR vs XDR in one line:**

> EDR = one flight recorder per plane. XDR = the whole air-traffic-control system watching every plane, tower, and radar feed together.

**Typical signal sources feeding XDR:**

- 🖥️ Endpoint (EDR data)
- 🌐 Network (IDS/IPS, firewall logs)
- ☁️ Cloud workloads (CSPM/cloud logs)
- 📧 Email security gateway
- 🪪 Identity provider (login anomalies, impossible travel)

---

## 📊 4. SIEM — Security Information & Event Management

> [!tip] TL;DR The **central log warehouse + correlation engine** for an entire organization — not just security tools, but firewalls, servers, apps, cloud services, everything that produces logs.

**Core jobs of a SIEM:**

- 📥 **Ingest** logs from every system (endpoints, servers, firewalls, apps, cloud, EDR/XDR)
- 🔗 **Correlate** events across sources using detection rules ("if X from firewall + Y from EDR happen within 5 min → alert")
- 🚨 **Alert** the SOC when a rule/pattern matches
- 🗄️ **Retain** logs long-term for compliance & forensic investigation (this is the piece that ties directly into **SOC 2 / ISO 27001** audit evidence — logging & monitoring is a control area in both)

**SIEM vs XDR — aren't they the same thing?**

||SIEM|XDR|
|---|---|---|
|Scope|Everything with a log (broad, shallow-ish)|Security-relevant telemetry (narrower, deeper)|
|Strength|Compliance, long retention, custom rules|Pre-built, high-fidelity threat detection|
|Setup effort|High (you write/tune detection rules)|Lower (vendor pre-builds detections)|
|Modern trend|Many orgs feed XDR output _into_ the SIEM|Often the "detection brain" behind SOC alerts|

---

## 🤖 5. SOAR — Security Orchestration, Automation & Response

> [!tip] TL;DR Takes the **manual steps** a human analyst would do after an alert fires, and automates them into a repeatable **playbook**.

**What a SOAR playbook might automate, step by step:**

1. 🚨 Alert fires in SIEM: "suspicious login from new country"
2. 🔍 SOAR auto-enriches: pulls user's recent login history, checks IP reputation
3. ❓ SOAR checks: is MFA enabled for this user? Is this a known VPN exit node?
4. 🚫 If confidence is high → SOAR **auto-disables the account** and opens a ticket
5. 📩 If confidence is low → SOAR **notifies an analyst** with all context pre-gathered

**Why it matters:**

- ⏱️ Cuts response time from hours to seconds for well-understood scenarios
- 🧑‍💻 Frees analysts from repetitive triage so they can focus on real investigation
- 📈 Scales a small SOC team's effective coverage

---

## 🏢 6. SOC — Security Operations Center

> [!tip] TL;DR The **team and process** (not a tool) that watches SIEM/XDR alerts, runs SOAR playbooks, and makes the human judgment calls a machine can't.

**Typical SOC tiers:**

|Tier|Role|Focus|
|---|---|---|
|🥉 Tier 1|Triage analyst|First look at alerts, filter false positives, escalate real ones|
|🥈 Tier 2|Incident responder|Deep investigation, containment actions|
|🥇 Tier 3|Threat hunter / lead|Proactive hunting, tunes detection rules, handles the worst incidents|
|🛠️ Detection engineer|Writes/tunes SIEM & EDR rules|Keeps false-positive rate manageable|

**SOC models:**

- 🏠 **In-house SOC** — your own team, 24/7 or business-hours only
- 🌍 **MSSP** (Managed Security Service Provider) — outsourced SOC-as-a-service
- 🔀 **Hybrid** — in-house team + MSSP for off-hours coverage

---

## 🔄 7. Putting It All Together — A Mini Incident Walkthrough

> [!example] Scenario: an employee's laptop gets infected via a phishing email
> 
> 1. 📧 Email gateway flags a suspicious attachment → logged to SIEM
> 2. 👤 User opens it anyway → **AMSI** flags a malicious PowerShell payload trying to run
> 3. 🖥️ **EDR** records the full process chain (Outlook → Word → PowerShell → outbound connection) even though AMSI blocked execution
> 4. 🔗 **XDR** correlates: this endpoint activity + the earlier flagged email + an unusual login from the same user's account minutes later
> 5. 📊 **SIEM** rule fires: "malicious script attempt + suspicious login within 10 minutes, same user" → high-severity alert
> 6. 🤖 **SOAR** playbook auto-isolates the laptop from the network and disables the user's session token
> 7. 🏢 **SOC Tier 1** picks up the pre-enriched alert, confirms it's real, escalates to **Tier 2** for full investigation and remediation

---

## 🧠 Quick Review (Self-Quiz)

> [!question]- What's the core difference between EDR and antivirus/HIPS? AV/HIPS block known-bad behavior in the moment. EDR continuously records telemetry so analysts can detect, investigate, and respond to things that got past prevention — over a longer time window.

> [!question]- In one sentence, how is XDR different from EDR? XDR extends EDR's correlation approach beyond the endpoint to include network, cloud, email, and identity signals in one unified view.

> [!question]- What's the main strength of a SIEM that XDR doesn't usually replace? Broad log ingestion from _everything_ (not just security-relevant telemetry) plus long-term retention needed for compliance and forensic audits (e.g., SOC 2 / ISO 27001 evidence).

> [!question]- Give an example of a task a SOAR playbook could fully automate. Enriching a suspicious-login alert with IP reputation and login history, then auto-disabling the account if confidence is high — without a human touching it first.

> [!question]- What are the three typical SOC analyst tiers, from first-look to deepest investigation? Tier 1 (triage) → Tier 2 (incident response/containment) → Tier 3 (threat hunting / rule tuning).

> [!question]- In the mini incident walkthrough, which component made the "isolate the laptop" decision automatically, and which component made the final human judgment call? SOAR automated the isolation action; the SOC (Tier 1 → Tier 2 analysts) made the human judgment calls.

---

🔗 Related: [[Endpoint-Security-Components]]