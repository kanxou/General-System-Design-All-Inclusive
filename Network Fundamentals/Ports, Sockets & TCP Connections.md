## Overview

When a client communicates with a server (HTTP, HTTPS, WebSocket, etc.), the communication happens over **TCP connections**.

To understand this, you need to know four concepts:

* **Process (PID)**
* **Port**
* **Socket**
* **TCP Connection**

---

# Process (PID)

A **Process ID (PID)** identifies a running program on the operating system.

Example:

```text
Nginx

PID = 2481
```

The PID is **not** used for networking. It simply identifies the running process.

---

# Port

A **port** is a **16-bit number (0–65535)** used by the operating system to determine **which application should receive incoming network traffic**.

Think of it as the "door number" of a service.

Common ports:

| Port | Service     |
| ---- | ----------- |
| 80   | HTTP        |
| 443  | HTTPS / WSS |
| 22   | SSH         |
| 3306 | MySQL       |
| 6379 | Redis       |

A server application asks the OS:

```text
Bind me to port 80.
```

From then on, the OS sends all traffic arriving on port 80 to that application.

---

# PID vs Port

| PID                          | Port                         |
| ---------------------------- | ---------------------------- |
| Identifies a running process | Identifies a network service |
| Managed by OS                | Managed by TCP/UDP           |
| Example: 2481                | Example: 80                  |

Relationship:

```text
Port 80
    │
    ▼
Nginx (PID 2481)
```

---

# Socket

A **socket** is an **operating system object** that represents one endpoint of network communication.

Conceptually, it stores information like:

```text
Socket

Source IP

Source Port

Destination IP

Destination Port

TCP State

Send Buffer

Receive Buffer
```

Every TCP connection has a socket on the client and another on the server.

---

# Types of Sockets

## 1. Listening Socket

Created when a server starts.

```text
socket()

↓

bind(80)

↓

listen()
```

State:

```text
LISTEN
```

Purpose:

* Wait for new TCP connections.
* Never carries HTTP/WebSocket data.

There is typically **one listening socket per listening port**.

---

## 2. Connected Socket

Created after a client successfully connects.

```text
Client

↓

TCP Handshake

↓

accept()

↓

Connected Socket
```

This socket is used to exchange HTTP requests, WebSocket frames, etc.

Each connected client gets its own connected socket.

---

# Complete Connection Flow

## Step 1 - Server Starts

```text
Nginx

↓

bind(port 80)

↓

listen()
```

OS creates:

```text
Listening Socket

203.0.113.10:80

State = LISTEN
```

The server is now waiting for connections.

---

## Step 2 - Client Connects

Browser opens:

```text
http://example.com
```

OS chooses a temporary client port.

Example:

```text
Client

192.168.1.5:54321
```

Connection request:

```text
192.168.1.5:54321

↓

203.0.113.10:80
```

---

## Step 3 - TCP Handshake

```text
Client        Server

SYN  ─────────►

     ◄──────── SYN-ACK

ACK  ─────────►
```

Connection established.

---

## Step 4 - accept()

The OS creates a **new connected socket**.

```text
Listening Socket

Port 80

↓

accept()

↓

Connected Socket
```

Important:

The listening socket **continues listening** for new clients.

---

## Step 5 - HTTP Communication

The client sends:

```http
GET /users HTTP/1.1
```

This request travels over the **connected socket**, not the listening socket.

---

## Step 6 - Close

When communication finishes:

```text
Connected Socket

↓

Closed
```

The listening socket remains active.

---

# Does the Server Create a New Port?

**No.**

Only a **new socket** is created.

Example:

```text
Listening Socket

203.0.113.10:80

↓

Connected Socket

203.0.113.10:80
```

Notice the server port remains **80**.

No new port is allocated.

---

# How Can One Port Handle Thousands of Clients?

Each connection is uniquely identified by the TCP **4-tuple**:

```text
(Source IP,
 Source Port,
 Destination IP,
 Destination Port)
```

Example:

Client A

```text
10.0.0.5:51001

↓

203.0.113.10:80
```

Client B

```text
10.0.0.6:51002

↓

203.0.113.10:80
```

Both use destination port **80**, but the source ports are different.

Therefore, they are different TCP connections.

---

# Why Does the Client Use Large Port Numbers?

The operating system automatically chooses an available **ephemeral (temporary) port**.

Examples:

```text
49152

52311

60012
```

These ports exist only for that connection and are automatically released when the connection closes.

---

# HTTP Example

```text
Listening Socket

203.0.113.10:80

        │

        ├──── Connected Socket A
        │
        │ GET /users
        │
        ├──── Connected Socket B
        │
        │ GET /products
        │
        └──── Connected Socket C
             GET /orders
```

The listening socket continues accepting new clients while each connected socket handles one TCP connection.

---

# WebSocket Example

WebSocket begins as HTTP.

```text
Client

↓

HTTP GET + Upgrade

↓

101 Switching Protocols

↓

WebSocket
```

The TCP connection does **not** change.

Before upgrade:

```text
Client:54321

↓

Server:443
```

After upgrade:

```text
Client:54321

↓

Server:443
```

Same ports.

Same socket.

Only the application protocol changes from HTTP to WebSocket.

---

# Important Terms

| Term             | Meaning                                                                                         |
| ---------------- | ----------------------------------------------------------------------------------------------- |
| PID              | Running process identifier                                                                      |
| Port             | Network endpoint number used by a service                                                       |
| Listening Socket | Waits for new TCP connections                                                                   |
| Connected Socket | Handles communication with one client                                                           |
| TCP Connection   | Communication channel between client and server                                                 |
| Ephemeral Port   | Temporary client-side port assigned by the OS                                                   |
| 4-Tuple          | (Source IP, Source Port, Destination IP, Destination Port) uniquely identifies a TCP connection |

---

# Key Takeaways

* A **port** is just a number that identifies a network service.
* A **PID** identifies a running process, not a network endpoint.
* A **socket** is a kernel object representing one endpoint of communication.
* A **listening socket** waits for new TCP connections.
* Every accepted client gets its own **connected socket**.
* The server **does not create a new port** for each client—every connected socket can still use the same server port (e.g., 80 or 443).
* Multiple clients can connect to the same server port simultaneously because each TCP connection is uniquely identified by its **4-tuple**.
* HTTP requests and WebSocket messages are exchanged over **connected sockets**, never over the listening socket.
