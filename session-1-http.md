# Session 1 — HTTP Protocol

## What is HTTP?

* HTTP (HyperText Transfer Protocol) is the medium through which the browser (client) talks to the server
* It is a set of rules for how a request is sent and how a response comes back
* Every time you open a website, submit a form, or an app loads data, an HTTP request is being made

## Ideas at the heart of HTTP

* **Stateless**
   * The server has no memory of past interactions
   * Each request is self-contained and carries all the info the server needs (who you are, what you want, in what format)
   * Example: if you make 2 requests one after another, the server treats them as 2 strangers
   * Benefit: simplicity
      * The server does not have to store anything about the client between requests
      * So any request can go to any server, which makes it easy to add more servers (scalability)
   * Downside: to "remember" a logged-in user, we need extra mechanisms like cookies, sessions, or tokens (covered in Session 3)

* **Client–Server Model**
   * Communication is always started by the client
   * The server only responds, it never starts a conversation on its own
   * Client = browser, mobile app, another server
   * Server = the machine that holds data / runs business logic

* **Transport underneath**
   * HTTP/1.1 and HTTP/2 run on top of TCP
      * TCP is reliable: guarantees data arrives, in order, without loss
   * HTTP/3 runs on QUIC, which is built on top of UDP
      * UDP is faster but does not guarantee delivery by itself
      * QUIC adds reliability on top, and makes connections faster to set up

## HTTP Messages

* Two types of messages
   * **Request** — sent by client
      * Method (GET, POST…)
      * URL / path (`/users/42`)
      * Headers
      * Body (optional — e.g. form data, JSON)
   * **Response** — sent by server
      * Status code (200, 404…)
      * Headers
      * Body (HTML, JSON, image…)

* **Headers** — extra info about the request/response, written as `Key: Value`
   * **General headers**
      * Apply to the message as a whole
      * e.g. `Date`, `Connection`, `Cache-Control`
   * **Request headers**
      * Info about the client / what it wants
      * e.g. `User-Agent` (which browser), `Accept` (what format it wants), `Authorization` (credentials)
   * **Representation headers**
      * Describe the body being sent
      * e.g. `Content-Type: application/json`, `Content-Length`, `Content-Encoding: gzip`
   * **Security headers**
      * Tell the browser how to behave safely
      * e.g. `Strict-Transport-Security` (always use HTTPS), `Content-Security-Policy` (which scripts are allowed), `X-Frame-Options` (block embedding in iframes)

## Methods

* A method tells the server the **intent** of the request

| Method | Intent | Example |
|---|---|---|
| GET | Read data | Get a user's profile |
| POST | Create new data | Create a new user |
| PUT | Replace data fully | Replace the whole profile |
| PATCH | Update part of data | Change only the email |
| DELETE | Remove data | Delete the user |
| OPTIONS | Ask server what is allowed | Used in CORS preflight |

* **Idempotent vs Non-Idempotent**
   * Idempotent = doing it once or doing it 10 times gives the same end result on the server
   * Idempotent: GET, PUT, DELETE
      * GET 10 times → nothing changes
      * PUT same data 10 times → same final state
      * DELETE 10 times → still deleted (first one deletes, rest do nothing)
   * Non-idempotent: POST (and usually PATCH)
      * POST 10 times → 10 new records created
      * PATCH like "increase count by 1" run 10 times → count goes up by 10
   * Why it matters: if a network request fails, it is safe to retry idempotent requests automatically, but retrying POST could create duplicates (e.g. double payment)

## CORS (Cross-Origin Requests)

* **Origin** = protocol + domain + port
   * `https://app.com` and `https://api.app.com` are different origins
* By default, browsers block a web page from calling a different origin (for security)
* CORS is how the server says "yes, this other origin is allowed to call me"
* Two kinds of cross-origin requests
   * **Simple request**
      * GET / POST / HEAD with basic headers and simple content types (like form data)
      * Browser sends it directly
      * Server replies with `Access-Control-Allow-Origin` header
      * If the header doesn't allow the origin, browser blocks the response
   * **Preflight request**
      * Happens for "non-simple" requests — e.g. PUT, DELETE, custom headers, `Content-Type: application/json`
      * Browser first sends an **OPTIONS** request to ask: "Can I send this method with these headers?"
      * Server replies with what it allows (`Access-Control-Allow-Methods`, `Access-Control-Allow-Headers`)
      * Only if allowed, the browser sends the actual request
      * So one action = 2 requests (can be cached with `Access-Control-Max-Age` to avoid repeating)

## Status Codes

* 3-digit number telling the result of the request

| Range | Meaning | Common ones |
|---|---|---|
| 1xx | Informational | 101 Switching Protocols |
| 2xx | Success | 200 OK, 201 Created, 204 No Content |
| 3xx | Redirect | 301 Moved Permanently, 302 Found, 304 Not Modified |
| 4xx | Client made a mistake | 400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found, 429 Too Many Requests |
| 5xx | Server failed | 500 Internal Server Error, 502 Bad Gateway, 503 Service Unavailable |

* 401 vs 403
   * 401 → "I don't know who you are" (not logged in)
   * 403 → "I know who you are, but you're not allowed"

## Caching

* Storing a copy of a response so it doesn't have to be fetched again
* Saves time and server load
* Controlled with headers
   * `Cache-Control: max-age=3600` → reuse this for 1 hour
   * `Cache-Control: no-store` → never cache (e.g. bank details)
   * `ETag` → a fingerprint of the content
      * Next time, client asks "has it changed since this ETag?" (`If-None-Match`)
      * If not changed → server replies `304 Not Modified` with no body → saves bandwidth
   * `Last-Modified` / `If-Modified-Since` → same idea but using time

## Content Negotiation

* Client and server agreeing on the best format for the response
* Client says what it can accept
   * `Accept: application/json` → format
   * `Accept-Language: en, hi` → language
   * `Accept-Encoding: gzip, br` → compression
* Server picks the best match and tells what it sent via `Content-Type`, `Content-Language`, `Content-Encoding`
* If server can't give any acceptable format → `406 Not Acceptable`

## HTTPS (quick note)

* HTTPS = HTTP + encryption (TLS)
* Nobody in between (Wi-Fi owner, ISP) can read or change the data
* Needs an SSL/TLS certificate on the server — tools like Certbot get free ones from Let's Encrypt
