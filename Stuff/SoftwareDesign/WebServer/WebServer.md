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
## 🖥️ What Is a Web Server?

A **web server** is both hardware and software that delivers web content (HTML pages, images, videos, APIs) to clients (usually web browsers) over the Internet. When you visit a website, your browser sends an HTTP request to the web server, which processes it and responds with the requested resources.

---
## 🔧 How to Use a Web Server

1. **Install** the server software on your machine or cloud instance.
2. **Configure** settings via configuration files (e.g., `nginx.conf`, `.htaccess`, `httpd.conf`).
3. **Deploy** your website files (HTML, CSS, JS, assets) into the server’s “document root.”
4. **Start** the server service (e.g., `systemctl start nginx`, `apachectl start`).
5. **Test** by navigating to `http://your-server-ip` or your domain name.
6. **Maintain & Monitor** logs and performance metrics to ensure uptime and security.
---
## 🌐 Popular Web Servers

### 🚀 [[Nginx]]

> Pronounced “engine‑ex,” [[Nginx]] is known for its high performance and low memory footprint.

- **✅ Pros**
    - 🌩️ Exceptional at handling high concurrency (many users at once)
    - ⚙️ Lightweight and efficient event‑driven architecture
    - 🔄 Built‑in load balancing and reverse proxy features
- **❌ Cons**
    - 🛠️ Slightly steeper learning curve for complex configurations
    - 📜 Less mature .htaccess‑style per-directory configuration

---
### 🐘 Apache HTTP Server

> The world’s most popular web server, famous for flexibility and a vast ecosystem of modules.

- **✅ Pros**
    - 🧩 Modular design (extensive third‑party modules)
    - 🔧 Granular per-directory (​.htaccess) configuration
    - 🤝 Excellent community support and documentation
- **❌ Cons**
    - 🐢 Higher memory usage under very heavy loads
    - 🔄 Prefork mode can be less efficient than event‑driven servers
---
### 🪟 Microsoft IIS

> A Windows‑based server integrated tightly with the Microsoft ecosystem.

- **✅ Pros**
    - 🔒 Seamless Windows authentication and Active Directory integration
    - 🛠️ User‑friendly GUI management via IIS Manager
    - 📦 .NET framework support out of the box
- **❌ Cons**
    - 💵 Licensing costs for Windows Server
    - 🌐 Less common on Linux/Unix environments
---
### 🏎️ LiteSpeed

> A commercial server known for Apache‑compatibility and high performance.

- **✅ Pros**
    - ⚡ Drop‑in replacement for Apache (compatible with .htaccess)
    - 🚀 Built‑in cache acceleration for dynamic content
    - 🔐 Good security features and DDoS protections
- **❌ Cons**
    - 💰 Commercial licensing (open‑source edition limited)
    - 🛠️ Smaller community than Apache or Nginx
---
### 💧 Caddy

> An easy‑to‑use server with automatic HTTPS via Let’s Encrypt.

- **✅ Pros**
    - 🔐 Automatic TLS certificates and renewal
    - 📝 Simple, human‑friendly configuration file (Caddyfile)
    - 🔌 Plugin architecture for extensibility
- **❌ Cons**
    - 📦 Less battle‑tested in very large deployments
    - ⚙️ Fewer modules/plugins than Apache or Nginx
---
### 💡 Lighttpd

> A lightweight server optimized for speed‐critical environments.

- **✅ Pros**
    - 🏃 Super low memory footprint
    - 🌐 Event‑driven (good for static content)
    - 🔨 Minimalist design—easy to secure
- **❌ Cons**
    - 🔌 Fewer features out of the box (less module support)
    - 🧩 Smaller community and ecosystem
---
### 🎉 Choosing the Right Server

- **Static Sites / High Concurrency:** Nginx, Lighttpd
- **Dynamic Content / Legacy Apps:** Apache, LiteSpeed
- **Windows‑Centric Environments:** IIS
- **Simplicity & HTTPS‑Ready:** Caddy
---
