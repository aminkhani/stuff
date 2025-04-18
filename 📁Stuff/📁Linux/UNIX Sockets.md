```table-of-contents
```
---
## 🧱 What Are Unix Domain Sockets?

Unix Domain Sockets are like **local pipelines** that let programs on the same machine communicate **without using the network stack**.

> Think of them as **fast, secure communication tunnels** between apps on the same server.

---
## 🌟 Why Use Unix Sockets?

| Advantage            | Description                               |
| -------------------- | ----------------------------------------- |
| ⚡ Faster             | No TCP/IP overhead — pure local comms     |
| 🔒 More Secure       | Not exposed to the network                |
| 📂 File-Based        | Identified by pathnames like `/tmp/sock`  |
| 🧠 Simple Permission | Use Linux file permissions (chmod, chown) |

---
## 📦 File Location Example

When you create a Unix socket, it appears like a file:

```
/tmp/my_socket.sock  🧱
```

You can `ls` or `lsof` it like any other file.

---
## 🔧 Code Example: Server & Client

### 👨‍💻 Server

```c
// 1️⃣ Create socket
int sockfd = socket(AF_UNIX, SOCK_STREAM, 0);
struct sockaddr_un addr;
addr.sun_family = AF_UNIX;
strcpy(addr.sun_path, "/tmp/mysock");

// Unlink any old socket file
unlink("/tmp/mysock");

// 2️⃣ Bind
bind(sockfd, (struct sockaddr*)&addr, sizeof(addr));

// 3️⃣ Listen
listen(sockfd, 5);

// 4️⃣ Accept
int client = accept(sockfd, NULL, NULL);

// 5️⃣ Read
read(client, buffer, 1024);

// 6️⃣ Respond
write(client, "Hello from server!", 19);
```
### 👩‍💻 Client

```c
int sockfd = socket(AF_UNIX, SOCK_STREAM, 0);
struct sockaddr_un addr;
addr.sun_family = AF_UNIX;
strcpy(addr.sun_path, "/tmp/mysock");

connect(sockfd, (struct sockaddr*)&addr, sizeof(addr));
write(sockfd, "Hi server!", 10);
read(sockfd, buffer, 1024);
```
---
## 🔍 Inspecting UDS Files

You can check Unix socket activity with:

```bash
# Lists all Unix domain sockets lsof | grep mysock
ss -x      
```

---
## 🧪 Common Use Cases

|App / Service|Usage|
|---|---|
|PostgreSQL|Default communication method 🔧|
|Nginx ↔ PHP-FPM|Uses UDS for fast handoff ⚙️|
|Docker & systemd|Internal daemons talk over UDS 🔁|

---
## ⚠️ Gotchas

- 🔧 You must `unlink()` the socket file if it already exists before binding.
- 🧑‍🔧 Permissions can block access — remember to `chmod` the socket file.
- 🛑 No remote access — only works locally.
---
## 🧾 Summary

- 🧱 **Unix Sockets** are fast, secure IPC endpoints for local communication
- 📂 Use file paths like `/tmp/sock` instead of IP:Port
- ✅ Great for performance-critical or secure environments
- 🔧 Use `AF_UNIX` instead of `AF_INET` in your code
---
