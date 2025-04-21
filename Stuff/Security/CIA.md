Three core principles of **information security**
![[attachments/Pasted image 20250421124726.png]]

### 🔒 Confidentiality  

Ensures that **sensitive information** is only accessible to those who are authorized.
#### What it means
- Preventing unauthorized disclosure of data.
#### How it’s enforced
- Encryption, access controls, strong authentication.
#### Example
- A company encrypts customer records so that even if someone intercepts the database, they can’t read the personal details without the decryption key.
---
### 🛡️ Integrity  

Ensures that **data remains accurate and unaltered**, except by authorized actions.
#### What it means
- Preventing unauthorized modification or corruption of data.
#### How it’s enforced
- Checksums, digital signatures, version control.
#### Example
- When you download software, the publisher provides a SHA‑256 hash. You compute the hash on your copy and compare it—if it matches, you know the file hasn’t been tampered with.
---
### ⚡ Availability

Ensures that information and systems are accessible when needed by authorized users.

#### What it means
- Preventing downtime or denial of service.
#### How it’s enforced
- Redundancy, backups, fault-tolerant design, DDoS protection.
#### Example
- An online store runs its website on multiple servers in different data centers. If one server goes down, traffic is automatically rerouted so customers can still make purchases.
---
### 🏥 Putting It All Together: A Hospital Records Scenario

Imagine a hospital’s electronic health record (EHR) system:

1. **🔒 Confidentiality:** Only doctors and nurses can view patient charts—patients’ lab results are encrypted in the database.
2. **🛡️ Integrity:** Every time a lab result is entered, a digital signature and timestamp are attached. Any alteration would break the signature, alerting staff to potential tampering.
3. **⚡ Availability:** The EHR runs across multiple servers with real‑time replication and battery‑backed power supplies, so doctors can access records even during a power outage or hardware failure.

By balancing 🔒, 🛡️, and ⚡, the hospital protects patient privacy, ensures accurate records, and keeps critical systems up and running.

---