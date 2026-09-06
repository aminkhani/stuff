## 🗺️ 1. The Big Picture — Which Framework Answers Which Question

```
 "How does an attack actually unfold, step by step?"      →  🔪 Cyber Kill Chain
 "What specific techniques do real attackers use?"        →  🗺️ MITRE ATT&CK
 "How do we structure an incident's evidence?"            →  💎 Diamond Model
 "Should we trust this request just because it's inside
  our network?"                                           →  🚫 Zero Trust
 "How do we manage security as an ongoing risk program?"  →  🔁 NIST CSF
 "How do we prove our controls to an auditor?"             →  📋 SOC 2 / ISO 27001
```

> [!success] One-line memory hook **Kill Chain & Diamond Model explain an _attack_. ATT&CK catalogs _techniques_. Zero Trust is an _architecture principle_. NIST CSF & SOC2/ISO27001 are _management/compliance_ programs.**

---

## 🔪 2. Cyber Kill Chain

> [!tip] TL;DR A linear, 7-stage model (originally by Lockheed Martin) describing how a typical intrusion unfolds — the core idea: **break any one link, and the attack fails.**

|Stage|Emoji|What happens|
|---|---|---|
|1. Reconnaissance|🔍|Attacker researches the target (OSINT, LinkedIn, DNS records)|
|2. Weaponization|🛠️|Builds the malicious payload (e.g., malware-laced PDF)|
|3. Delivery|📧|Sends it — phishing email, USB drop, watering-hole site|
|4. Exploitation|💥|Payload executes, exploits a vulnerability|
|5. Installation|📦|Malware/backdoor installs persistence on the host|
|6. Command & Control (C2)|🤖📡|Malware phones home to attacker infrastructure|
|7. Actions on Objectives|🎯|Data theft, ransomware, lateral movement, sabotage|

> [!warning] Known limitation The Kill Chain is **linear** and was built around traditional malware delivery. It fits phishing/malware scenarios well but doesn't map cleanly onto insider threats, cloud misconfig attacks, or supply-chain compromises — that's part of why ATT&CK (below) became popular alongside it.

---

## 💎 3. Diamond Model of Intrusion Analysis

> [!tip] TL;DR A simple 4-point model for describing **any single malicious event**, used heavily in threat intelligence write-ups.

```
        Adversary
       /          \
Capability ---- Victim
       \          /
       Infrastructure
```

- 👤 **Adversary** — who's doing it
- 🛠️ **Capability** — what tool/malware/technique they used
- 🖥️ **Victim** — who/what was targeted
- 🌐 **Infrastructure** — what servers/domains/IPs they used to deliver or control it

> [!note] Why it's useful Unlike the linear Kill Chain, the Diamond Model is meant to be applied **per-event** and then chained together (event 1's infrastructure might become event 2's victim data source) — good for connecting multiple incidents into one campaign.

---

## 🗺️ 4. MITRE ATT&CK

> [!tip] TL;DR A giant, continuously updated **knowledge base of real-world attacker Tactics, Techniques, and Procedures (TTPs)** — think of it as the periodic table of "things attackers actually do."

**Structure:**

- 🎯 **Tactics** — the _why_ (the attacker's goal at that stage) — e.g., "Initial Access," "Privilege Escalation," "Exfiltration"
- 🔧 **Techniques** (and sub-techniques) — the _how_ — e.g., under "Initial Access" → "Phishing" → sub-technique "Spearphishing Attachment"
- 📝 **Procedures** — the specific real-world implementation a particular threat group used

**Tactics, roughly in order (this is the "Enterprise ATT&CK Matrix"):** Reconnaissance → Resource Development → Initial Access → Execution → Persistence → Privilege Escalation → Defense Evasion → Credential Access → Discovery → Lateral Movement → Collection → C2 → Exfiltration → Impact

**Why it matters practically:**

- 🎯 Detection engineers map their SIEM/EDR rules to specific ATT&CK techniques → shows **coverage gaps** ("we detect Technique X but have zero coverage for Technique Y")
- 🔴 Red teams / pentesters plan exercises using real ATT&CK technique IDs (e.g., **T1059** = Command and Scripting Interpreter)
- 🧩 Vendors advertise "MITRE ATT&CK coverage" as a selling point for EDR/XDR products

> [!note] Kill Chain vs ATT&CK Kill Chain = 7 broad linear stages (simple, high-level). ATT&CK = hundreds of specific, non-linear techniques attackers can mix and match at each stage. Most teams use ATT&CK for day-to-day detection work and Kill Chain for quick executive-level storytelling.

---

## 🚫 5. Zero Trust

> [!tip] TL;DR An architecture **principle**, not a single product: _"Never trust, always verify"_ — no request is automatically trusted just because it came from inside the corporate network.

**Core principles:**

- 🔐 **Verify explicitly** — authenticate & authorize every request, every time, using all available signals (identity, device health, location)
- 🎚️ **Least-privilege access** — grant only the minimum access needed, just-in-time where possible
- 🧱 **Assume breach** — design as if an attacker is already inside; segment aggressively so a breach in one area can't spread everywhere

**How it differs from the old "castle-and-moat" model:**

|Castle-and-Moat (perimeter)|Zero Trust|
|---|---|---|
|Trust basis|"Inside the network = trusted"|Trust nothing by default, verify continuously|
|Access control|Broad, network-location-based (VPN in = full access)|Narrow, identity + context-based, per-request|
|Blast radius if breached|Large (attacker moves freely once inside)|Small (micro-segmentation limits lateral movement)|

**Related pieces you'll see mentioned alongside Zero Trust:**

- 🧩 **ZTNA** (Zero Trust Network Access) — modern app-level access instead of flat VPN
- 🪪 **MFA / continuous authentication** — supports "verify explicitly"
- 🔬 **Micro-segmentation** — supports "assume breach"

---

## 🔁 6. NIST Cybersecurity Framework (CSF)

> [!tip] TL;DR A high-level **risk management framework** (from the U.S. National Institute of Standards and Technology) organizing all of security into 5 (now 6, in CSF 2.0) core functions — used to structure a whole security _program_, not just one control.

**The core functions:**

|Function|Emoji|Question it answers|
|---|---|---|
|**Govern** _(added in CSF 2.0)_|🏛️|Do we have policy, risk strategy, and accountability in place?|
|**Identify**|🔎|What assets, data, and risks do we actually have?|
|**Protect**|🛡️|What safeguards do we put in place? _(this is where IDP/HIPS/AMSI/MFA live)_|
|**Detect**|👀|How do we notice something bad is happening? _(EDR/XDR/SIEM live here)_|
|**Respond**|🚨|What do we do once we've detected it? _(SOAR/SOC/IR live here)_|
|**Recover**|🔄|How do we restore normal operations afterward?|

> [!success] Why this ties your whole learning path together Every note you've made so far slots into this framework:
> 
> - [[Endpoint-Security-Components]] (IDP/HIPS/AMSI) → **Protect**
> - [[Detection-and-Response]] (EDR/XDR/SIEM/SOAR/SOC) → **Detect** + **Respond**
> - This note's Zero Trust → mostly **Protect**, with a bit of **Identify**
> - MITRE ATT&CK → used across **Detect** (rule coverage) and **Identify** (risk assessment)

---

## 📋 7. Where SOC 2 / ISO 27001 Fit In

> [!tip] TL;DR These aren't attack/defense frameworks — they're **compliance/audit frameworks** that prove _to a third party_ (a customer, a regulator) that your security program is actually functioning.

- 📄 **SOC 2** — an audit report (Type I = point-in-time, Type II = over a period) against 5 "Trust Service Criteria": Security, Availability, Processing Integrity, Confidentiality, Privacy
- 🌍 **ISO 27001** — an international certifiable standard for an **Information Security Management System (ISMS)** — broader and more prescriptive than SOC 2

**Practical link to everything above:** an auditor checking SOC 2/ISO 27001 will often literally ask "show me your logging/monitoring" (→ SIEM), "show me your incident response process" (→ SOC/SOAR), "show me your endpoint protection" (→ IDP/HIPS/AMSI) — the frameworks above are the _proof_ behind those audit checkboxes.

---

## 🧠 Quick Review (Self-Quiz)

> [!question]- What's the key difference between the Cyber Kill Chain and MITRE ATT&CK? Kill Chain is a simple, linear 7-stage model of how an attack generally unfolds. ATT&CK is a much larger, non-linear catalog of hundreds of specific real-world techniques attackers use, organized under broader tactics.

> [!question]- Name the four points of the Diamond Model. Adversary, Capability, Infrastructure, Victim.

> [!question]- What does "assume breach" mean in Zero Trust, and what design principle supports it? Design your network as if an attacker is already inside, so a compromise in one place can't spread everywhere. Micro-segmentation is the principle that supports this.

> [!question]- In NIST CSF, which function would EDR/XDR/SIEM fall under, and which would SOAR/SOC fall under? EDR/XDR/SIEM → **Detect**. SOAR/SOC (incident handling) → **Respond**.

> [!question]- What's an ATT&CK "Tactic" vs a "Technique"? Give an example of each. A Tactic is the attacker's _goal_ (e.g., "Initial Access"). A Technique is the specific _method_ used to achieve it (e.g., "Phishing" or its sub-technique "Spearphishing Attachment").

> [!question]- What's the practical difference between SOC 2 and ISO 27001? SOC 2 is an audit report (Type I/II) against 5 Trust Service Criteria, mostly used in the US/vendor-assessment context. ISO 27001 is an internationally certifiable standard for a full Information Security Management System (ISMS) — broader and more prescriptive.

---

🔗 Related: [[Endpoint-Security-Components]] · [[Detection-and-Response]]