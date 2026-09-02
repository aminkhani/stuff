## NIST Cybersecurity Framework (CSF)

The NIST CSF is a voluntary framework by the National Institute of Standards and Technology to help organizations manage and reduce cybersecurity risk. **CSF 2.0** (released Feb 2024) added a new function.

### Core Functions

- **Govern** – Establish risk management strategy, roles, policy, and oversight (new in 2.0)
- **Identify** – Understand assets, data, and risks (asset inventory, risk assessment)
- **Protect** – Implement safeguards (access control, encryption, secure config, training)
- **Detect** – Continuous monitoring to spot anomalies and incidents
- **Respond** – Contain and mitigate detected incidents
- **Recover** – Restore capabilities and services after an incident

### Structure

Each function breaks into **Categories** → **Subcategories** → **Informative References** (mapped to standards like ISO 27001, CIS Controls, SOC 2).

### Implementation Tiers

| Tier | Description |
|------|-------------|
| 1 – Partial | Ad hoc, reactive |
| 2 – Risk Informed | Risk-aware but not org-wide |
| 3 – Repeatable | Formalized, consistent policies |
| 4 – Adaptive | Proactive, continuously improving |

### Profiles

"Current" vs "Target" profiles let you gap-assess where you are vs. where you want to be.

### Relevance for Django/Backend Security Work

- **Protect**: maps well to things like `SECURE_*` settings, CSRF/CORS config, secrets management, dependency scanning (`pip-audit`, `safety`)
- **Detect**: logging/monitoring (Sentry, structured logs, WAF alerts)
- **Identify**: data flow mapping — what PII touches your Django models/DB
- **Respond/Recover**: incident runbooks, backup/restore strategy for your DB layer
