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

|Domain|Meaning|Emoji|
|---|---|---|
|`AF_INET`|IPv4 Internet|🌍|
|`AF_INET6`|IPv6 Internet|🌐|
|`AF_UNIX`|Unix local socket|🧱|

---
## 🔧 Socket Types

|Constant|Meaning|Icon|
|---|---|---|
|`SOCK_STREAM`|TCP (reliable) 📦|✅ Ordered|
|`SOCK_DGRAM`|UDP (fast) ⚡|📡 No order|
|`SOCK_RAW`|Raw sockets 🛠️|🐾 Low-level|

---
## 🔐 Bonus: Common Syscalls & Tools

|Tool / Call|Purpose|Icon|
|---|---|---|
|`netstat`, `ss`|View active sockets 📊|📡 Connections|
|`lsof -i`|List open files (incl. sockets) 📂|🧰 Debugging|
|`select`, `poll`|Monitor multiple sockets ⏲️|👀 Async I/O|
|`recv`, `send`|For reading/writing data ✉️|📤📥 Messaging|

---
## 📦 Socket File Example ([[🧱 UNIX Sockets]])

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
