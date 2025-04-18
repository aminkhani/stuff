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
## 🧩 What Is a Socket?

A **socket** is like a **virtual plug & cable** 🧷🔌 that allows two programs to talk to each other — whether on the same computer 🖥️ or across the internet 🌐.

> 🗣️ **In short:**  
> A **socket = communication endpoint** used for sending and receiving data.

---
## 📬 Types of Sockets in Linux

|Type|Description|Icon & Use Case|
|---|---|---|
|**Stream (TCP)**|Reliable, connection-based 📦➡️📦|Web servers, file transfers, etc.|
|**Datagram (UDP)**|Connectionless, fast, no guarantee ⚡📤|DNS, online games, video streaming|
|**Raw Sockets**|For low-level access 📡⚙️|Custom protocols, packet sniffers|
|**Unix Domain Sockets**|Local machine-only 🧱🖥️|Fast IPC (inter-process communication)|

---
## 🛠️ Socket Workflow in C (TCP Example)

Let’s walk through how a **server ↔ client** connection works using sockets.
### 👨‍💻 Server-Side

```c
// 1️⃣ Create socket
int sockfd = socket(AF_INET, SOCK_STREAM, 0);

// 2️⃣ Bind to IP/port
bind(sockfd, ...);

// 3️⃣ Listen for connections
listen(sockfd, 5);

// 4️⃣ Accept a client
int client = accept(sockfd, ...);

// 5️⃣ Read data
read(client, buffer, 1024);

// 6️⃣ Respond
write(client, "Hello", 5);
```
### 👩‍💻 Client-Side

```c
// 1️⃣ Create socket
int sockfd = socket(AF_INET, SOCK_STREAM, 0);

// 2️⃣ Connect to server
connect(sockfd, ...);

// 3️⃣ Send message
write(sockfd, "Hi", 2);

// 4️⃣ Read reply
read(sockfd, buffer, 1024);
```
---
## 🌐 Socket Domains

| Domain     | Meaning           |
| ---------- | ----------------- |
| `AF_INET`  | IPv4 Internet     |
| `AF_INET6` | IPv6 Internet     |
| `AF_UNIX`  | Unix local socket |

---
## 🔧 Socket Types

| Constant      | Meaning           |
| ------------- | ----------------- |
| `SOCK_STREAM` | TCP (reliable) 📦 |
| `SOCK_DGRAM`  | UDP (fast) ⚡      |
| `SOCK_RAW`    | Raw sockets 🛠️   |

---
## 🔐 Bonus: Common Syscalls & Tools

| Tool / Call      | Purpose                            |
| ---------------- | ---------------------------------- |
| `netstat`, `ss`  | View active sockets 📊             |
| `lsof -i`        | List open files (incl. sockets) 📂 |
| `select`, `poll` | Monitor multiple sockets ⏲️        |
| `recv`, `send`   | For reading/writing data ✉️        |

---
## 📦 Socket File Example ([[UnixSockets]])

If you use a UNIX domain socket, it creates a file on disk:

```bash
/tmp/my_socket.sock  🧱
```

Applications can use this file to talk directly, **faster than TCP**.

---
## 🧪 Debugging Tip

Use `netcat` or `telnet` to test socket connections:

```bash
# Server nc localhost 1234  # Client
nc -l 1234
```
---
