# HTTP (Hypertext Transfer Protocol) — Complete Notes

## What is HTTP?

HTTP (Hypertext Transfer Protocol) is an **application layer**,
**stateless**, **client-server** protocol used for communication between
clients (browser/mobile app) and servers.

- Request → Response model
- Runs over TCP (HTTP/1.x, HTTP/2)
- Runs over QUIC (UDP) in HTTP/3
- Defined originally by RFC 2616, now superseded by **RFC 9110–9114** (2022), which split HTTP into: Semantics (9110), Caching (9111), HTTP/1.1 (9112), HTTP/2 (9113), HTTP/3 (9114).

---

# HTTP Architecture

```text
Client
   │
   │ HTTP Request
   ▼
Server
   │
   │ HTTP Response
   ▼
Client
```

A more complete real-world picture usually includes intermediaries:

```text
Client → DNS Resolver → Client
Client → TLS Handshake → Server
Client → CDN / Reverse Proxy → Load Balancer → App Server → Database
```

**Key definitions**
- **Client**: the entity initiating a request (browser, mobile app, another server, CLI tool like `curl`).
- **Server**: the entity that listens on a port and returns a response.
- **Origin**: the combination of scheme + host + port (e.g. `https://api.example.com:443`) that defines a "same-origin" boundary.
- **Proxy**: an intermediary that forwards requests on behalf of a client (forward proxy) or a server (reverse proxy).
- **Gateway**: a server that translates between protocols (e.g. HTTP to gRPC).

---

# HTTP Request

An HTTP request consists of:

- Request Line
- Headers
- Body (optional)

```http
GET /users/123 HTTP/1.1
Host: api.example.com
Authorization: Bearer abc123
Accept: application/json
```

### Request Line

```text
GET /users/123 HTTP/1.1
│      │           │
│      │           └── HTTP Version
│      └────────────── URI / Resource
└───────────────────── Method
```

### Request Target Forms
- **Origin form**: `/users/123` (most common, used with `Host` header)
- **Absolute form**: `http://api.example.com/users/123` (used by proxies)
- **Authority form**: `api.example.com:443` (used with `CONNECT`)
- **Asterisk form**: `*` (used with `OPTIONS *`)

---

# HTTP Response

An HTTP response consists of:

- Status Line
- Headers
- Body

```http
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 30

{
    "id":123,
    "name":"John"
}
```

### Status Line

```text
HTTP/1.1 200 OK
│           │    │
│           │    └── Reason Phrase (human readable, optional in HTTP/2+)
│           └─────── Status Code
└─────────────────── HTTP Version
```

---

# URL vs URI vs URN

- **URI (Uniform Resource Identifier)** — identifies a resource. Umbrella term.
- **URL (Uniform Resource Locator)** — a URI that also tells you *how to locate* the resource (scheme + host).
- **URN (Uniform Resource Name)** — a URI that names a resource without specifying location, e.g. `urn:isbn:0451450523`.

```text
URI
 ├── URL: https://api.example.com/users/123
 └── URN: urn:isbn:0451450523
```

Every URL is a URI. Not every URI is a URL.

### Anatomy of a URL

```text
https://user:pass@api.example.com:8443/users/123?active=true#profile
│      │              │            │      │           │        │
scheme  userinfo      host        port    path       query   fragment
```

---

# Important Headers

## Request Headers

```text
Host
Authorization
Accept
Accept-Encoding
Accept-Language
User-Agent
Cookie
If-None-Match
If-Modified-Since
Content-Type
Content-Length
Referer
Origin
X-Forwarded-For
X-Request-ID
```

## Response Headers

```text
Content-Type
Content-Length
Cache-Control
Set-Cookie
Location
ETag
Last-Modified
Transfer-Encoding
Vary
Retry-After
```

## Security Headers (commonly asked, often missing from intro notes)

```text
Strict-Transport-Security   → forces HTTPS for future requests (HSTS)
Content-Security-Policy     → restricts sources of scripts/styles/images (mitigates XSS)
X-Content-Type-Options: nosniff → stops browsers from MIME-sniffing
X-Frame-Options              → mitigates clickjacking (superseded by CSP frame-ancestors)
Referrer-Policy               → controls how much referrer info is sent
Access-Control-Allow-Origin   → CORS: which origins may read the response
Permissions-Policy            → controls browser feature access (camera, geolocation, etc.)
```

---

# Stateless Protocol

HTTP itself stores **no client state** between requests.

Each request is independent.

State is maintained using:

- Cookies
- Session IDs
- JWT
- Database
- Redis
- OAuth tokens

**Why stateless?** Statelessness simplifies server design and scalability — any server in a pool can handle any request, since no session affinity is required at the protocol level (though applications may still choose "sticky sessions" for performance).

---

# HTTP Methods

| Method  | Purpose                                       | Safe | Idempotent |
|---------|------------------------------------------------|------|------------|
| GET     | Retrieve resource                              | ✅   | ✅ |
| POST    | Create subordinate resource / process request  | ❌   | ❌ |
| PUT     | Replace entire resource                        | ❌   | ✅ |
| PATCH   | Partial update                                 | ❌   | Usually ❌ |
| DELETE  | Delete resource                                | ❌   | ✅ |
| HEAD    | Same as GET without body                       | ✅   | ✅ |
| OPTIONS | Discover supported methods                     | ✅   | ✅ |
| TRACE   | Diagnostic loopback                            | ✅   | ✅ |
| CONNECT | Create proxy tunnel                            | ❌   | ❌ |

> **QUERY** is a proposed (not yet standardized in RFC 9110) method meant to send a body with a safe, cacheable, read-only request — solving the long-standing "GET with a body" problem. As of early 2026 it remains a draft.

---

# Safe vs Idempotent vs Cacheable

## Safe
Does not modify server state (read-only). Safe methods can be crawled, prefetched, or cached without side effects.

```text
GET
HEAD
OPTIONS
TRACE
```

## Idempotent
Performing the request multiple times produces the same final server state as performing it once.

```text
PUT
DELETE
GET
HEAD
OPTIONS
```

Example:

```http
DELETE /users/5
```

First call deletes the user. Second call may return `404`. Final server state remains the same. **Note: POST is neither safe nor idempotent by default** — this is why browsers warn "Are you sure you want to resubmit this form?" on refresh.

## Cacheable

Not every safe/idempotent method is cacheable by default.

```text
GET     → cacheable by default
HEAD    → cacheable by default
POST    → cacheable only if explicit freshness/caching headers are present
```

---

# HTTP Status Codes

## 1xx — Informational

```text
100 Continue
101 Switching Protocols
102 Processing (WebDAV)
103 Early Hints
```

## 2xx — Success

```text
200 OK
201 Created
202 Accepted
203 Non-Authoritative Information
204 No Content
205 Reset Content
206 Partial Content
```

## 3xx — Redirection

```text
300 Multiple Choices
301 Moved Permanently
302 Found (Temporary Redirect)
303 See Other
304 Not Modified
307 Temporary Redirect (method preserved)
308 Permanent Redirect (method preserved)
```

> **301 vs 308 / 302 vs 307**: The newer codes (307/308) guarantee the HTTP method and body are preserved on redirect; 301/302 historically allowed clients to change POST → GET.

## 4xx — Client Errors

```text
400 Bad Request
401 Unauthorized        (missing/invalid authentication)
403 Forbidden           (authenticated but not permitted)
404 Not Found
405 Method Not Allowed
406 Not Acceptable
408 Request Timeout
409 Conflict
410 Gone
415 Unsupported Media Type
422 Unprocessable Content
425 Too Early
428 Precondition Required
429 Too Many Requests
431 Request Header Fields Too Large
```

> **401 vs 403** is a very common interview question: 401 means "who are you?" (authentication failure), 403 means "I know who you are, but you can't do this" (authorization failure).

## 5xx — Server Errors

```text
500 Internal Server Error
501 Not Implemented
502 Bad Gateway
503 Service Unavailable
504 Gateway Timeout
505 HTTP Version Not Supported
507 Insufficient Storage
```

---

# HTTPS

HTTPS = HTTP + TLS

Provides:

- **Encryption** — data can't be read in transit
- **Authentication** — server identity is verified via certificates (and optionally client, via mTLS)
- **Integrity** — data can't be tampered with undetected

TLS handshake occurs before HTTP messages are exchanged.

### TLS Handshake (simplified, TLS 1.3)

```text
Client Hello  → (supported ciphers, key share)
Server Hello  ← (chosen cipher, certificate, key share)
                [Finished — encrypted application data starts]
```

TLS 1.3 reduced the handshake to **1-RTT** (down from 2-RTT in TLS 1.2), and supports **0-RTT resumption** for repeat connections — this is a key enabler for HTTP/3's fast reconnects.

---

# Authentication

```text
Authorization: Bearer <JWT>

Authorization: Basic <base64 of user:pass>

Cookie: sessionId=...

Authorization: Digest ...   (challenge-response, rarely used today)
```

### Common Auth Schemes
- **Session-based**: server stores session state, client holds an opaque cookie ID.
- **Token-based (JWT)**: server issues a signed, stateless token; server verifies signature on each request without a DB lookup.
- **OAuth 2.0**: delegated authorization framework (e.g. "Sign in with Google") — issues access & refresh tokens.
- **API Keys**: static secret sent via header, simpler but less secure than OAuth/JWT.
- **mTLS (mutual TLS)**: both client and server present certificates — common in service-to-service auth.

---

# Cookies

Server sends:

```text
Set-Cookie: sessionId=abc123; HttpOnly; Secure; SameSite=Strict; Max-Age=3600
```

Browser automatically sends:

```text
Cookie: sessionId=abc123
```

### Cookie Attributes
```text
HttpOnly   → JS cannot access via document.cookie (mitigates XSS theft)
Secure     → only sent over HTTPS
SameSite   → Strict / Lax / None — controls cross-site sending (mitigates CSRF)
Max-Age / Expires → lifetime of the cookie
Domain / Path      → scope of the cookie
```

---

# CORS (Cross-Origin Resource Sharing)

Missing from many intro notes but a frequent real-world/interview topic.

- Browsers block cross-origin requests by default (Same-Origin Policy).
- CORS is a set of response headers that relax this restriction.
- **Simple requests** (GET/POST with basic headers) go straight through with `Access-Control-Allow-Origin` checked on response.
- **Preflighted requests** (custom headers, non-simple methods like PUT/DELETE, `Content-Type: application/json`) trigger an `OPTIONS` request first:

```http
OPTIONS /users/123 HTTP/1.1
Origin: https://myapp.com
Access-Control-Request-Method: PUT
Access-Control-Request-Headers: Content-Type
```

```http
HTTP/1.1 204 No Content
Access-Control-Allow-Origin: https://myapp.com
Access-Control-Allow-Methods: GET, POST, PUT
Access-Control-Allow-Headers: Content-Type
Access-Control-Max-Age: 600
```

---

# Content Negotiation

Client:

```text
Accept: application/json
Accept-Encoding: gzip, br
Accept-Language: en-US
Accept-Charset: utf-8
```

Server:

```text
Content-Type
Content-Encoding
Content-Language
Vary: Accept-Encoding    (tells caches the response varies by this request header)
```

---

# Caching

Important headers:

```text
Cache-Control
Expires
ETag
If-None-Match
Last-Modified
If-Modified-Since
Age
```

### Cache-Control Directives
```text
no-store        → never cache
no-cache        → cache but revalidate every time
private         → only browser cache, not shared/CDN cache
public          → can be cached by any cache including CDNs
max-age=3600    → freshness lifetime in seconds
must-revalidate → once stale, must revalidate before use
immutable       → resource will never change (great for hashed static assets)
```

### Validation flow (conditional requests)

```text
ETag: "abc123"

If-None-Match: "abc123"
```

Server replies:

```text
304 Not Modified   (no body sent — saves bandwidth)
```

**ETag vs Last-Modified**: ETag is a content hash/fingerprint (precise), Last-Modified is a timestamp (coarser, second-level precision). ETag is generally preferred when available.

---

# HTTP Versions

## HTTP/0.9 (1991)

- GET only
- HTML only
- No headers
- No status codes
- One TCP connection per request

## HTTP/1.0 (1996)

- POST
- HEAD
- Status codes
- Request/Response headers
- Multiple content types
- Content-Length
- New TCP connection for every request

## HTTP/1.1 (1997)

Features:

- Persistent Connections (`Connection: keep-alive`)
- Host header (Virtual Hosting)
- Chunked Transfer Encoding
- Pipelining (rarely used)
- Standardized PUT, DELETE, OPTIONS, TRACE
- Range requests (`Range`, `Content-Range`) for partial downloads

Problems:

- HTTP Head-of-Line (HOL) blocking — responses must return in the order requested on a connection
- No multiplexing — browsers open multiple TCP connections per origin (typically 6) as a workaround
- No header compression — headers (including cookies) repeated on every request, wasting bandwidth
- Text protocol — parsing overhead, ambiguity risks (e.g. request smuggling)
- Domain sharding — sites split assets across subdomains just to get more parallel connections

---

## HTTP/2 (2015) — Deep Dive

Based on Google's **SPDY** protocol. Standardized as RFC 7540 (now RFC 9113).

### Core Concepts

**1. Binary Framing Layer**
Instead of plain text, HTTP/2 breaks messages into binary **frames** (HEADERS, DATA, SETTINGS, WINDOW_UPDATE, RST_STREAM, PING, GOAWAY, etc.), all multiplexed onto a single TCP connection.

```text
Frame = [Length | Type | Flags | Stream ID | Payload]
```
Frame 1: [Len=50] [Type=HEADERS] [Flags=END_HEADERS] [StreamID=1] → headers for /style.css
Frame 2: [Len=30] [Type=HEADERS] [Flags=END_HEADERS] [StreamID=3] → headers for /app.js
Frame 3: [Len=1200][Type=DATA]    [Flags=0]            [StreamID=1] → body chunk for /style.css
Frame 4: [Len=800] [Type=DATA]    [Flags=0]            [StreamID=3] → body chunk for /app.js
Frame 5: [Len=200] [Type=DATA]    [Flags=END_STREAM]   [StreamID=1] → last chunk, /style.css done

**2. Streams and Multiplexing**
A single TCP connection carries multiple **streams**, each an independent, bidirectional sequence of frames identified by a Stream ID. Requests/responses on different streams can be interleaved — no more "one request at a time per connection."

```text
Connection
 ├── Stream 1: GET /style.css
 ├── Stream 3: GET /app.js
 └── Stream 5: GET /data.json
   (all in flight simultaneously on ONE TCP connection)
```

**3. Stream Prioritization**
Clients can express a dependency tree and weight for streams, hinting the server which resources matter more (e.g. render-blocking CSS before images). In practice, browser support for this was inconsistent, and it's one reason HTTP/2 prioritization was largely reworked in later specs.

**4. HPACK Header Compression**
Headers are repetitive across requests (cookies, user-agent, etc). HPACK maintains a shared, indexed table of previously-sent header fields (both a static table and a dynamic table built per-connection) so repeated headers are sent as small index references instead of full text — dramatically shrinking overhead.

**5. Flow Control**
Per-stream and per-connection flow control windows (`WINDOW_UPDATE` frames) prevent a fast sender from overwhelming a slow receiver, similar in spirit to TCP flow control but at the application layer.

**6. Server Push (now deprecated/removed from Chrome in 2022)**
Allowed a server to proactively send resources (like CSS/JS) before the client asked for them, anticipating what the HTML would require. Removed largely because: it was hard to avoid pushing resources the client already had cached, cache coordination was error-prone, and gains in practice were smaller/more inconsistent than hoped. Modern alternative: **`103 Early Hints`**, which lets a server tell the client which resources to start fetching while the server is still preparing the main response.

**7. Single Connection Per Origin**
Because multiplexing removes the need for 6 parallel TCP connections, HTTP/2 typically uses just one long-lived TCP+TLS connection per origin — reducing handshake and slow-start overhead.

### Remaining Problem: TCP Head-of-Line Blocking
Even though HTTP/2 eliminated *application-layer* HOL blocking, all streams still share one **TCP** connection. If a single TCP packet is lost, TCP's in-order delivery guarantee stalls *every* stream until that packet is retransmitted — even streams whose data already arrived. This transport-layer HOL blocking is the core motivation for HTTP/3.

---

## HTTP/3 (2022) — Deep Dive

Standardized as RFC 9114. Built on **QUIC** (RFC 9000), which itself runs over **UDP**.

### Why UDP + QUIC instead of TCP?
TCP's in-order, single-stream-of-bytes delivery is baked into the OS kernel and can't be changed without new kernels rolling out everywhere — a practically impossible upgrade path at internet scale. QUIC reimplements reliability and congestion control **in user space, on top of UDP**, so it can evolve independently of OS kernels and browsers can ship updates directly.

### Core Concepts

**1. Independent Streams at the Transport Layer**
Unlike HTTP/2 (where multiplexing is an HTTP-layer concept riding on one TCP byte-stream), QUIC implements multiplexed streams **natively in the transport**. Packet loss on one QUIC stream only stalls that stream — other streams keep flowing. This eliminates transport-layer HOL blocking.

```text
QUIC Connection (over UDP)
 ├── Stream A: lost packet → only Stream A stalls
 ├── Stream B: unaffected, keeps delivering
 └── Stream C: unaffected, keeps delivering
```

**2. TLS 1.3 Built-In**
QUIC integrates the TLS 1.3 handshake directly into its connection setup handshake — there's no separate "TCP handshake, then TLS handshake" sequence. This collapses connection setup from 2-3 round trips down to effectively **1-RTT**, and supports **0-RTT** for resumed connections (client can send data immediately using a cached key from a prior session, at the cost of some replay-attack considerations for non-idempotent requests).

**3. Connection Migration**
A QUIC connection is identified by a **Connection ID**, not by the traditional TCP 4-tuple (source IP, source port, dest IP, dest port). This means if a client's IP changes — e.g. switching from WiFi to mobile data — the QUIC connection survives without renegotiation. TCP connections break in this scenario and must be fully re-established.

**4. QPACK Header Compression**
QUIC's equivalent of HPACK, redesigned to avoid a subtle issue: HPACK's dynamic table updates must be processed in strict order, which would reintroduce HOL blocking across streams. QPACK decouples header compression state from strict ordering using dedicated unidirectional streams, preserving HTTP/2-level compression efficiency without sacrificing independent stream delivery.

**5. Improved Congestion Control**
Because QUIC is implemented in user space rather than the OS kernel, it can iterate faster on congestion control algorithms (e.g. BBR) and ship improvements via browser/library updates rather than OS updates.

**6. No Server Push**
HTTP/3 drops Server Push entirely, leaning on `103 Early Hints` instead.

### Trade-offs of HTTP/3
- **UDP can be blocked or throttled** by some corporate firewalls/middleboxes that don't recognize QUIC traffic, requiring fallback to HTTP/2.
- **More CPU overhead** on servers in some cases, since encryption/congestion-control work formerly done by the kernel is now done in user space.
- **Adoption is still uneven**: as of the last update to these notes, most major CDNs (Cloudflare, Google, Fastly) and browsers support HTTP/3, but plenty of origin servers still default to HTTP/2 or HTTP/1.1.

---

# HTTP Version Comparison

| Feature                        | HTTP/1.0 | HTTP/1.1         | HTTP/2           | HTTP/3      |
|---------------------------------|----------|------------------|------------------|-------------|
| Persistent connections          | ❌       | ✅               | ✅               | ✅          |
| Pipelining                      | ❌       | ✅ (rarely used) | N/A              | N/A         |
| Multiplexing                    | ❌       | ❌               | ✅ (per TCP conn)| ✅ (native) |
| Binary protocol                 | ❌       | ❌               | ✅               | ✅          |
| Header compression              | ❌       | ❌               | HPACK            | QPACK       |
| Server Push                     | ❌       | ❌               | ✅ (deprecated)  | ❌          |
| Transport                       | TCP      | TCP              | TCP              | QUIC (UDP)  |
| HTTP-layer HOL blocking         | N/A      | Yes              | No               | No          |
| Transport-layer HOL blocking    | N/A      | Yes              | Yes (TCP)        | No          |
| Built-in TLS                    | ❌       | ❌               | ❌ (separate)    | ✅ (integrated) |
| Connection migration            | ❌       | ❌               | ❌               | ✅          |
| Handshake RTT (cold start)      | High     | High             | Medium (TCP+TLS) | Low (1-RTT, 0-RTT resumption) |

---

# REST, Idempotency Keys, and Related Concepts

Often expected alongside HTTP knowledge in interviews:

- **REST (Representational State Transfer)**: an architectural style using HTTP methods/status codes/URIs to represent and manipulate resources statelessly.
- **Richardson Maturity Model**: Level 0 (single endpoint, POST everything) → Level 1 (multiple resource URIs) → Level 2 (proper use of HTTP verbs/status codes) → Level 3 (HATEOAS — responses include links to related actions).
- **Idempotency Keys**: for non-idempotent methods like POST (e.g. payment creation), clients send a unique `Idempotency-Key` header so retries don't create duplicate resources — a common pattern in payment APIs (Stripe, etc.).
- **Rate Limiting**: often implemented via `429 Too Many Requests` + `Retry-After` header, using token bucket or sliding window algorithms.
- **Load Balancing basics**: reverse proxies (NGINX, HAProxy, ALB) distribute requests across backend servers using round-robin, least-connections, or consistent hashing strategies; `X-Forwarded-For` preserves original client IP through the chain.

---

# HTTP vs WebSockets vs gRPC (frequently confused)

| Protocol   | Model                          | Transport | Use case |
|------------|----------------------------------|-----------|----------|
| HTTP       | Request-response                 | TCP/QUIC  | Typical web APIs |
| WebSocket  | Full-duplex, persistent connection | TCP (upgraded from HTTP via `101 Switching Protocols`) | Real-time chat, live feeds |
| SSE (Server-Sent Events) | One-way server → client streaming over plain HTTP | TCP | Live notifications, dashboards |
| gRPC       | RPC calls, uses HTTP/2 framing under the hood | TCP (HTTP/2) | Internal microservice-to-microservice communication |

---

# Common Interview Questions

- Difference between PUT and PATCH
- Why is HTTP stateless?
- Why is GET safe?
- What is idempotency? Is POST idempotent?
- URL vs URI vs URN
- HTTP vs HTTPS
- 401 vs 403 — what's the difference?
- Why HTTP/2? What problem does multiplexing solve?
- Why HTTP/3? What is transport-layer HOL blocking and how does QUIC fix it?
- What are ETags, and how do they differ from Last-Modified?
- Chunked Transfer Encoding — what is it and when is it used?
- Cookies vs JWT — trade-offs of each
- Virtual Hosting — how does the Host header enable it?
- Multiplexing — HTTP/2 vs HTTP/3 implementation differences
- Content-Type vs Accept
- What is CORS and why does a preflight OPTIONS request happen?
- What is connection migration and why does QUIC support it but TCP doesn't?
- Explain the TLS 1.3 handshake and how HTTP/3 integrates it
