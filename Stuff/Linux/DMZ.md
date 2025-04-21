## 🌐 DMZ (Demilitarized Zone) in Networking

A DMZ is a subnetwork that sits **between** an **internal trusted network** and an **untrusted external network** (usually the Internet). It adds an extra layer of security by isolating servers that need to be publicly accessible—such as web, mail, or DNS servers—so that if they’re compromised, attackers can’t directly reach your core LAN.
### Purpose
- Expose public‑facing services without putting internal assets at risk.
- Limit what attackers can access if a DMZ host is breached.
### Typical Setup
1. **Firewall Rule Set A:** Internet ⇄ DMZ (allows HTTP/HTTPS, SMTP, DNS)
2. **Firewall Rule Set B:** DMZ ⇄ Internal LAN (often minimal or one‑way, e.g., allowing database traffic from web server to DB server only)
3. **Firewall Rule Set C:** Internet ⇄ Internal LAN (usually blocked entirely)
---
## 🐧 DMZ in Linux (Firewalls & iptables)

On a Linux gateway or firewall, you implement a DMZ by defining network interfaces and crafting rules—often with `iptables` or `firewalld`—to segregate zones.

#### Network Interfaces
    - `eth0` → Internet (untrusted)
    - `eth1` → DMZ network (e.g., 192.168.10.0/24)
    - `eth2` → Internal LAN (e.g., 10.0.0.0/24)
#### iptables Example

```bash
# Define zone interfaces
WAN_IF="eth0"
DMZ_IF="eth1"
LAN_IF="eth2"

# Flush old rules
iptables -F
iptables -t nat -F
iptables -t mangle -F

# Default policies: drop everything by default
iptables -P INPUT   DROP
iptables -P FORWARD DROP
iptables -P OUTPUT  ACCEPT

# Allow established traffic
iptables -A INPUT   -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
iptables -A FORWARD -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

# Internet → DMZ: allow web (80/443) and DNS (53)
iptables -A FORWARD -i $WAN_IF -o $DMZ_IF \
  -p tcp -m multiport --dports 80,443 -j ACCEPT
iptables -A FORWARD -i $WAN_IF -o $DMZ_IF \
  -p udp --dport 53 -j ACCEPT

# DMZ → LAN: allow only specific service (e.g., MySQL port 3306)
iptables -A FORWARD -i $DMZ_IF -o $LAN_IF \
  -p tcp --dport 3306 -j ACCEPT

# Drop any direct WAN → LAN traffic
iptables -A FORWARD -i $WAN_IF -o $LAN_IF -j DROP
```
This ensures your public servers in the DMZ can serve the Internet, your web servers can query the internal database, and nothing else traverses zones.

---
### 🔍 Why Use a DMZ?

- **Containment:** If a DMZ server is hacked, attackers remain “stuck” there.
- **Minimal Exposure:** Only essential ports/services are open externally.
- **Audit & Monitoring:** DMZ traffic is easier to log and inspect without flooding internal logs.
---
### 🏗️ Variations & Best Practices

- **Dual‑Firewall DMZ:** Two firewalls in series, with the DMZ sandwiched between, for defense‑in‑depth.
- **Zoning:** Further subdivide DMZ (e.g., public web, payment processing) with more granular rules.
- **Host‑Based Hardening:** Even in the DMZ, lock down servers: disable unnecessary services, enforce strict SSH access policies, and keep systems patched.
---
> [!INFO] By segregating your network into at least three zones—**Internet**, **DMZ**, and **Internal LAN**—and carefully crafting firewall rules on your Linux gateway, you greatly reduce the risk that a public‑facing compromise leads to a full‑blown breach of sensitive internal systems.

