```table-of-contents
```
---
## 🔍 Overview

Nginx is an open‑source, high‑performance web server and reverse proxy. Its hallmark is an **asynchronous, event‑driven** design that lets it handle thousands of connections with minimal resources.

---
## 🧩 Core Components

| Component               | Description                                                                                      |
| ----------------------- | ------------------------------------------------------------------------------------------------ |
| **Master Process**      | ▶️ Reads configuration, manages worker processes (start/stop/reload).                            |
| **Worker Processes**    | 🛠️ Handle actual client connections, I/O, and request processing.                               |
| **Event Module**        | 🌐 Uses epoll/kqueue to efficiently multiplex socket events (new connections, data ready, etc.). |
| **HTTP Module**         | 📡 Implements HTTP server features: request parsing, headers, URI routing, caching, etc.         |
| **Stream Module**       | 🔀 Handles TCP/UDP proxying (e.g., database proxies, game servers).                              |
| **Mail Module**         | ✉️ Proxy for SMTP, IMAP, POP3 protocols (e‑mail load balancing).                                 |
| **Core Module**         | 🧩 Provides base directives (process management, logging, variables).                            |
| **Third‑Party Modules** | 🔌 Extend functionality (e.g., Lua scripting, geo‑filtering, advanced WAF).                      |

---
## 📝 Example: Simple Reverse‑Proxy Setup

```nginx
# /etc/nginx/nginx.conf
user  www-data www-data;
worker_processes auto;
error_log  /var/log/nginx/error.log notice;
pid        /var/run/nginx.pid;

events {
    worker_connections 1024;
}

http {
    include       mime.types;
    default_type  application/octet-stream;
    sendfile        on;
    keepalive_timeout 65;

    # Upstream backend pool
    upstream backend_app {
        server 127.0.0.1:3000;
        server 127.0.0.1:3001;
    }

    server {
        listen       80;
        server_name  example.com;

        # Static files
        location /static/ {
            root /var/www/html;
            expires 30d;
        }

        # Proxy dynamic requests
        location / {
            proxy_pass         http://backend_app;
            proxy_http_version 1.1;
            proxy_set_header   Upgrade $http_upgrade;
            proxy_set_header   Connection keep-alive;
            proxy_set_header   Host $host;
            proxy_cache_bypass $http_upgrade;
        }
    }
}
```
- 👑 **Master/Worker:** `worker_processes auto;` lets Nginx spawn one worker per CPU core.
- 🔄 **Event Loop:** Efficiently handles thousands of clients with `worker_connections`.
- 🔀 **Upstream:** Defines a load‑balanced pool (`upstream backend_app`).
- 📥📤 **Proxying:** `proxy_pass` sends requests to your app servers, keeping headers intact.
- 📷 **Static Serving:** `/static/` is served directly from disk, bypassing proxy.
---
## 🚀 Nginx’s Areas of Expertise

- **High Concurrency 🔥**  
    Handles tens of thousands of simultaneous connections with low memory footprint.
- **Reverse Proxy & Load Balancing ⚖️**  
    Distributes traffic across backend servers, health‑checks, sticky sessions, session persistence.
- **HTTP Caching 🗄️**  
    Caches dynamic content at the edge, reducing load on origin servers.
- **TLS/SSL Termination 🔒**  
    Offloads HTTPS processing (certs, handshakes) from backend, supports HTTP/2 & TLS 1.3.
- **Streaming & WebSockets 📡**  
    Proxying for live media streams or bi‑directional WebSocket connections.
- **Mail Proxy Support ✉️**  
    Load balances mail servers (IMAP, SMTP, POP3).
- **Extensibility via Modules 🔌**  
    Community‑contributed modules: Lua scripting, geo‑IP, Web Application Firewall (ModSecurity).
---
## ⚖️ When to Use Nginx

- **Static‑heavy sites** (e.g., CDNs, media sites)
- **API gateways** & microservices front‑ends
- **SSL/TLS offload** for secure sites
- **Edge caching** to accelerate dynamic applications
---
## 🔐 Security & Hardening

- **TLS/SSL Best Practices** 🔒
    - `ssl_protocols TLSv1.2 TLSv1.3; ssl_ciphers ECDHE…; ssl_session_tickets off;`
- **Rate‑Limiting & DDoS Mitigation** 🚫
    - `limit_req_zone`, `limit_conn_zone`, `fail2ban` + `http_auth_request`.
- **WAF & ModSecurity** 🛡️
    - Integrate ModSecurity or NGINX App Protect for OWASP rules, SQLi/XSS blocking.
- **Headers & Policies**
    - `add_header X‑Frame‑Options SAMEORIGIN; Content‑Security‑Policy …; HSTS`.
---
