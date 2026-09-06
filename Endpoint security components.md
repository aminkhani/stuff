## 🌐 1. IDP / IDS — Intrusion Detection & Prevention

> [!tip] TL;DR **Watches network traffic** for known attack patterns and either **alerts** (Detection) or **blocks** (Prevention) them automatically.

- 🔤 **IDS** = Intrusion **Detection** System → detects and logs/alerts
- 🔤 **IPS** = Intrusion **Prevention** System → detects **and** actively blocks
- 🔤 **IDP** = Intrusion **Detection and Prevention** → the combined term (sometimes vendors just call the whole module "IDS" even though it prevents too)

**How it works:**

- 🔍 Signature-based detection (matches known attack fingerprints)
- 📊 Anomaly-based detection (flags unusual traffic patterns)
- 🚫 Can auto-block the offending IP/connection

**Typical things it catches:**

- Port scans 🔎
- Brute-force login attempts 🔑
- Known exploit payloads in network packets 💥
- Botnet command-and-control (C2) traffic 🤖

> [!note] Related but different **IDP** ≠ **Identity Provider** (IdP, used in SSO/SAML/OAuth). Same acronym, totally different world — context (network security vs. authentication) tells you which one is meant.

---

## 🖥️ 2. HIP / HIPS — Host-based Intrusion Prevention System

> [!tip] TL;DR Like IDS/IPS, but it watches **what happens on the machine itself** (processes, files, registry) instead of network traffic.

**What HIPS monitors on the host:**

- 🧩 Running processes & their behavior
- 📁 File system changes
- 🗝️ Registry key modifications
- 🧬 Behavioral/heuristic patterns (not just signatures)

**HIPS is usually the "umbrella" for several sub-features:**

|Sub-feature|What it does|
|---|---|
|🧨 **Exploit Blocker**|Hardens commonly-attacked apps (browsers, PDF readers, email clients, MS Office) against exploit techniques|
|🧠 **Advanced Memory Scanner**|Catches malware that hides via obfuscation/encryption and only "unpacks" itself in memory|
|🔬 **Deep Behavioral Inspection**|Watches full process behavior chains, not just single actions|
|🔒 **Self-Defense**|Protects the security product's _own_ processes/files/registry from being disabled by malware|
|💰 **Ransomware Shield**|Behavior-based layer specifically for ransomware-style file encryption patterns|

> [!warning] HIPS ≠ Firewall HIPS only watches processes running **inside the OS**. It is **not** a firewall and doesn't filter network traffic by itself — that's IDS/IPS's job.

**HIP vs HIDS naming note:** some vendors say "HIPS" (Prevention), others say "HIDS" (Detection only) — same host-layer concept, different enforcement level (just like IDS vs IPS above).

---

## 🧬 3. AMSI Protection — Antimalware Scan Interface

> [!tip] TL;DR A Windows API that lets **any script/macro content** get scanned by your antivirus **right before it runs** — even if it's generated dynamically in memory and never touches disk.

**Why AMSI matters:**

- 📜 Covers **PowerShell**, **VBScript**, **JScript**, **.NET**, Office **macros**, WMI
- 👻 Specifically designed to defeat **fileless malware** (never saved as a file → traditional AV scanning of disk files misses it)
- 🧅 Also defeats **multi-layer obfuscation** — because AMSI scans the _de-obfuscated_ content right before execution, not the obfuscated source

**How it fits the pipeline:**

```
Script/macro is about to execute
        ↓
AMSI hands the (de-obfuscated) buffer to the registered AV engine
        ↓
AV engine returns a verdict (clean / malicious)
        ↓
Execution allowed ✅ or blocked 🚫
```

> [!danger] Attacker's counter-move: "AMSI Bypass" A common red-team/malware technique is patching or disabling AMSI in memory before running malicious script content. This is exactly why security products add a dedicated **"AMSI Protection"** toggle — it protects the AMSI interface itself from being tampered with, and flags processes that try to bypass it.

---

## 🧰 4. Other Common Endpoint Security Components

> [!tip] TL;DR These are the modules you'll usually see sitting _alongside_ IDP/HIPS/AMSI in the same product.

|Component|Emoji|What it protects against|
|---|---|---|
|**Real-time File System Protection**|📂|Classic on-access/on-write file scanning|
|**Network Attack Protection**|🌐🚧|Blocks exploit attempts at the network protocol level (sits close to IDS/IPS)|
|**Botnet Protection**|🤖⛓️|Detects C2 communication patterns even from unknown malware|
|**Application Control / Allowlisting**|📋✅|Only permits pre-approved apps to run|
|**Device Control**|🔌🚫|Restricts USB drives, external media, peripherals|
|**DLP (Data Loss Prevention)**|📤🔒|Stops sensitive data from leaving the org (email, USB, upload)|
|**Sandboxing / Detonation Chamber**|🧪📦|Runs suspicious files in an isolated VM to observe behavior safely|
|**EDR (Endpoint Detection & Response)**|🕵️‍♂️📡|Continuous telemetry + hunting + incident response, beyond just blocking|
|**Vulnerability/Patch Assessment**|🩹|Flags outdated/vulnerable software versions|
|**Web/URL Filtering**|🌍🚫|Blocks known-malicious or phishing domains|

---

## 🗺️ 5. Where Each Layer Sits (Mental Model)

```
🌐 NETWORK LAYER           🖥️ HOST / OS LAYER               📜 SCRIPT / MEMORY LAYER
───────────────             ─────────────────                ───────────────────────
IDS / IPS (IDP)       →     HIPS / HIDS                 →    AMSI Protection
Firewall                    Exploit Blocker                  Fileless-malware defense
Botnet Protection           Advanced Memory Scanner
Network Attack Prot.        Self-Defense / Ransomware Shield
```

> [!success] One-line memory hook **IDP watches the wire 🌐, HIPS watches the process 🖥️, AMSI watches the script right before it fires 📜.**

---

## 🧠 Quick Review (Self-Quiz)

> [!question]- What does IDS stand for, and how is it different from IPS? IDS = Intrusion **Detection** System (alerts only). IPS = Intrusion **Prevention** System (alerts **and** blocks). "IDP" usually means the combined detect+prevent capability.

> [!question]- Is HIPS a firewall? No. HIPS monitors processes, files, and registry keys **inside the OS** — it doesn't filter network traffic like a firewall does.

> [!question]- Name three sub-features usually bundled under HIPS. Exploit Blocker, Advanced Memory Scanner, Self-Defense, Deep Behavioral Inspection, Ransomware Shield (any three).

> [!question]- Why is AMSI especially good against fileless malware? Because it scans the script/code buffer **right before execution in memory**, so it doesn't matter that the malicious code was never written to disk.

> [!question]- What is an "AMSI bypass" and why does "AMSI Protection" exist? An AMSI bypass is when malware patches/disables the AMSI interface in memory so its script content never gets scanned. "AMSI Protection" is a dedicated defense that guards the AMSI interface itself from tampering.

> [!question]- Which layer does Botnet Protection belong to — network or host? Network layer — it looks at C2 communication patterns in traffic, similar in spirit to IDS/IPS.

> [!question]- What's the key difference between EDR and a traditional antivirus/HIPS setup? Antivirus/HIPS mainly _block_ known-bad behavior in real time. EDR adds continuous telemetry, historical visibility, and hunting/response capability — useful _after_ something slips through.

---