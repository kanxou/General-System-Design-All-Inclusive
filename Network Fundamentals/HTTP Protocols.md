# HTTP (Hypertext Transfer Protocol)

## What is HTTP?

HTTP (Hypertext Transfer Protocol) is an **application layer**,
**stateless**, **client-server** protocol used for communication between
clients (browser/mobile app) and servers.

-   Request → Response model
-   Runs over TCP (HTTP/1.x, HTTP/2)
-   Runs over QUIC (UDP) in HTTP/3

------------------------------------------------------------------------

# HTTP Architecture

``` text
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

------------------------------------------------------------------------

# HTTP Request

An HTTP request consists of:

-   Request Line
-   Headers
-   Body (optional)

``` http
GET /users/123 HTTP/1.1
Host: api.example.com
Authorization: Bearer abc123
Accept: application/json
```

### Request Line

``` text
GET /users/123 HTTP/1.1
│      │           │
│      │           └── HTTP Version
│      └────────────── URI / Resource
└───────────────────── Method
```

------------------------------------------------------------------------

# HTTP Response

An HTTP response consists of:

-   Status Line
-   Headers
-   Body

``` http
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 30

{
    "id":123,
    "name":"John"
}
```

------------------------------------------------------------------------

# URL vs URI

URI (Uniform Resource Identifier)

``` text
/users/123
```

URL (Uniform Resource Locator)

``` text
https://api.example.com/users/123
```

Every URL is a URI, but not every URI is a URL.

------------------------------------------------------------------------

# Important Headers

## Request Headers

``` text
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
```

## Response Headers

``` text
Content-Type
Content-Length
Cache-Control
Set-Cookie
Location
ETag
Last-Modified
Transfer-Encoding
```

------------------------------------------------------------------------

# Stateless Protocol

HTTP itself stores **no client state** between requests.

Each request is independent.

State is maintained using:

-   Cookies
-   Session IDs
-   JWT
-   Database
-   Redis
-   OAuth tokens

------------------------------------------------------------------------

# HTTP Methods

  Method    Purpose                                         Safe   Idempotent
  --------- ----------------------------------------------- ------ ------------
  GET       Retrieve resource                               ✅     ✅
  POST      Create subordinate resource / process request   ❌     ❌
  PUT       Replace entire resource                         ❌     ✅
  PATCH     Partial update                                  ❌     Usually ❌
  DELETE    Delete resource                                 ❌     ✅
  HEAD      Same as GET without body                        ✅     ✅
  OPTIONS   Discover supported methods                      ✅     ✅
  TRACE     Diagnostic loopback                             ✅     ✅
  CONNECT   Create proxy tunnel                             ❌     ❌

> **QUERY** is **not** a standard HTTP method in RFC 9110.

------------------------------------------------------------------------

# Safe vs Idempotent

## Safe

Does not modify server state.

    GET
    HEAD
    OPTIONS
    TRACE

## Idempotent

Performing the request multiple times produces the same final state.

    PUT
    DELETE
    GET
    HEAD
    OPTIONS

Example:

``` http
DELETE /users/5
```

First call deletes the user.

Second call may return `404`.

Final server state remains the same.

------------------------------------------------------------------------

# HTTP Status Codes

## 1xx --- Informational

    100 Continue
    101 Switching Protocols
    103 Early Hints

## 2xx --- Success

    200 OK
    201 Created
    202 Accepted
    204 No Content
    206 Partial Content

## 3xx --- Redirection

    301 Permanent Redirect
    302 Temporary Redirect
    304 Not Modified
    307 Temporary Redirect
    308 Permanent Redirect

## 4xx --- Client Errors

    400 Bad Request
    401 Unauthorized
    403 Forbidden
    404 Not Found
    405 Method Not Allowed
    409 Conflict
    415 Unsupported Media Type
    422 Unprocessable Content
    429 Too Many Requests

## 5xx --- Server Errors

    500 Internal Server Error
    501 Not Implemented
    502 Bad Gateway
    503 Service Unavailable
    504 Gateway Timeout

------------------------------------------------------------------------

# HTTPS

HTTPS = HTTP + TLS

Provides:

-   Encryption
-   Authentication
-   Integrity

TLS handshake occurs before HTTP messages are exchanged.

------------------------------------------------------------------------

# Authentication

``` text
Authorization: Bearer <JWT>

Authorization: Basic <base64>

Cookie: sessionId=...
```

------------------------------------------------------------------------

# Cookies

Server sends:

``` text
Set-Cookie:
```

Browser automatically sends:

``` text
Cookie:
```

------------------------------------------------------------------------

# Content Negotiation

Client:

``` text
Accept: application/json
Accept-Encoding: gzip
Accept-Language: en-US
```

Server:

``` text
Content-Type
Content-Encoding
```

------------------------------------------------------------------------

# Caching

Important headers:

``` text
Cache-Control
Expires
ETag
If-None-Match
Last-Modified
If-Modified-Since
```

Example:

``` text
ETag: abc123

If-None-Match: abc123
```

Server replies:

``` text
304 Not Modified
```

------------------------------------------------------------------------

# HTTP Versions

## HTTP/0.9 (1991)

-   GET only
-   HTML only
-   No headers
-   No status codes
-   One TCP connection per request

## HTTP/1.0 (1996)

-   POST
-   HEAD
-   Status codes
-   Request/Response headers
-   Multiple content types
-   Content-Length
-   New TCP connection for every request

## HTTP/1.1 (1997)

Features:

-   Persistent Connections (`Connection: keep-alive`)
-   Host header (Virtual Hosting)
-   Chunked Transfer Encoding
-   Pipelining (rarely used)
-   Standardized PUT, DELETE, OPTIONS, TRACE

Problems:

-   HTTP HOL blocking
-   No multiplexing
-   No header compression
-   Text protocol
-   Multiple TCP connections
-   Domain sharding

## HTTP/2 (2015)

Features:

-   Binary framing
-   Multiplexing
-   Streams
-   Flow control
-   Prioritization
-   HPACK header compression
-   Server Push (now deprecated)

Problem:

-   TCP Head-of-Line blocking remains.

## HTTP/3 (2022)

Built on QUIC over UDP.

Features:

-   Multiplexing
-   QPACK
-   0-RTT connection resumption
-   TLS 1.3 built-in
-   Connection migration
-   No transport-layer HOL blocking

------------------------------------------------------------------------

# HTTP Version Comparison

  Feature                        HTTP/1.0   HTTP/1.1           HTTP/2            HTTP/3
  ------------------------------ ---------- ------------------ ----------------- ------------
  Persistent connections         ❌         ✅                 ✅                ✅
  Pipelining                     ❌         ✅ (rarely used)   N/A               N/A
  Multiplexing                   ❌         ❌                 ✅                ✅
  Binary protocol                ❌         ❌                 ✅                ✅
  Header compression             ❌         ❌                 HPACK             QPACK
  Server Push                    ❌         ❌                 ✅ (deprecated)   ❌
  Transport                      TCP        TCP                TCP               QUIC (UDP)
  HTTP-layer HOL blocking        N/A        Yes                No                No
  Transport-layer HOL blocking   N/A        Yes                Yes (TCP)         No
  Built-in TLS                   ❌         ❌                 ❌                ✅

------------------------------------------------------------------------

# Common Interview Questions

-   Difference between PUT and PATCH
-   Why is HTTP stateless?
-   Why is GET safe?
-   What is idempotency?
-   URL vs URI
-   HTTP vs HTTPS
-   Why HTTP/2?
-   Why HTTP/3?
-   What are ETags?
-   Chunked Transfer Encoding
-   Cookies vs JWT
-   Virtual Hosting
-   Multiplexing
-   `Content-Type` vs `Accept`
