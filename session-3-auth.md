# Session 3 — Authentication & Authorization

* **Authentication** → *Who are you?*
* **Authorization** → *What are you allowed to do?*
* Authentication always comes first — you can't decide permissions without knowing who the user is

---

# Authentication

* Proving who you are in a given context (password, OTP, Google login, API key…)

## Sessions

* **Why sessions came**
   * HTTP is a stateless protocol — server forgets you after every request
   * This was fine for the early web, which was mostly static pages
   * With dynamic content (logins, carts, dashboards), statelessness became a bottleneck
   * The web needed stateful interaction → the concept of a session came
* **What is a session**
   * A way to keep temporary server-side context about a user
* **How a session is created**
   * User logs in
   * Server creates a random, unguessable `sessionId`
   * Server stores user info against this `sessionId` in a store
   * `sessionId` is sent to the user as a cookie
   * Every next request automatically carries this cookie
   * Server looks up the `sessionId` → knows who the user is
   * This gave the server a kind of "memory"
   * Sessions are short-lived and have an expiry (e.g. 30 min of inactivity)
* **Evolution of where sessions are stored**
   * Files on the server → slow, and breaks when you have more than one server
   * Database → shared across servers, but a DB lookup on every request is slow
   * **Redis** (in-memory store) → stores data in RAM instead of disk
      * RAM access is roughly 1000x faster than disk
      * Works as a shared store for all servers in a distributed setup

## JWT (JSON Web Token)

* **What led to JWT**
   * Storing millions of sessions in memory became costly
   * Keeping session data in sync across servers/regions added latency
* **What it is**
   * A stateless way to pass claims (info about the user) between two systems
   * Self-contained — the token itself carries the user data
   * Server doesn't need to look anything up, it just verifies the signature
* **Structure** — 3 parts joined by dots: `header.payload.signature`
   * Header → which algorithm is used (e.g. HS256)
   * Payload → user data / claims (`userId`, `role`, `exp` expiry)
   * Signature → made using a secret key only the server knows
      * If anyone changes the payload, the signature won't match → token rejected
* **Important:** payload is only encoded (base64), not encrypted
   * Anyone can decode and read it → never put passwords or secrets inside
* **Advantages**
   * Solved scalability and cost — no session store needed
   * Works well across multiple services
* **Disadvantages**
   * If someone steals your JWT, they can impersonate you until it expires
   * Hard to revoke — server can't "delete" a token it never stored
   * Common fix
      * Short-lived **access token** (e.g. 15 min)
      * Long-lived **refresh token** (e.g. 7 days) used to get a new access token and can be revoked

## Cookies

* A small piece of information stored on the client's browser by the server
* Once set, the browser automatically sends it on every next request to that same server
* Server sets it with a header
   * `Set-Cookie: sessionId=abc123; HttpOnly; Secure; SameSite=Strict; Max-Age=3600`
* Important flags
   * `HttpOnly` → JavaScript can't read it (protects against XSS stealing it)
   * `Secure` → only sent over HTTPS
   * `SameSite` → not sent on requests from other sites (protects against CSRF)
   * `Max-Age` / `Expires` → when it expires
* Cookies are the **storage/transport**; sessions or JWTs are **what's stored inside**

## Types of Authentication

* **Stateful (Session-based)**
   * User sends username & password
   * Server verifies and generates a `sessionId`
   * Saves user info in Redis
   * Sends `sessionId` back to client in a cookie
   * All next requests send this cookie
   * Server checks Redis and authenticates the user
   * ✅ Easy to revoke (just delete from Redis) ❌ needs a store

* **Stateless (JWT-based)**
   * Client sends username & password
   * Server verifies and generates a signed JWT containing user info
   * All next requests send the JWT in a header → `Authorization: Bearer <token>`
   * Server verifies the signature and authenticates the user
   * ✅ No store, scales easily ❌ hard to revoke

* **API Key**
   * A long random string generated once to give access
   * Sent with each request, e.g. `x-api-key: sk_live_...`
   * Easy to generate and use
   * Ideal for machine-to-machine communication (your server calling Stripe, OpenAI etc.)
   * Server should store only a **hash** of the key, not the key itself
   * Not meant for end users logging in

* **OAuth 2.0 / OpenID Connect**
   * **Why it was needed**
      * People reused the same password on many sites → one leak exposed everything
      * Password fatigue — too many accounts to remember
      * **Delegation problem** — one app needs data from another app
         * e.g. a social app wants to read your Gmail contacts to find friends
      * Sharing your Gmail password with that app is a huge security risk (it gets full access)
   * **Question:** how to give limited access without sharing passwords?
   * **OAuth 1.0** solved this by
      * Sharing **tokens with specific permissions** (scopes) instead of passwords
      * Defined 4 roles
         * Resource Owner → you (the user)
         * Client → the app asking for access
         * Resource Server → where your data lives (Gmail API)
         * Authorization Server → who issues tokens (Google's login)
      * Was complex to implement (every request needed cryptographic signing)
   * **OAuth 2.0**
      * Simpler — introduced **Bearer tokens** (whoever holds the token can use it, so always use HTTPS)
      * Came with different flows for different types of apps (web, mobile, devices, server-to-server)
      * **Authorization Code flow** (most common)
         * Client redirects user to the Auth Server (e.g. Google)
         * User logs in on Google and approves permissions ("Allow access to contacts")
         * Auth Server redirects back to client with a short-lived **authorization code**
         * Client's backend exchanges this code (+ its client secret) with the Auth Server for an **access token**
         * Client uses the access token to call the Resource Server (e.g. Google Keep / Contacts) on behalf of the user
      * Scopes ensure the client never gets more access than the user allowed
   * **OpenID Connect (OIDC)**
      * OAuth 2.0 is only about **authorization** (access to data), not about who the user is
      * OIDC is a layer on top that adds **authentication**
      * Along with the access token, it returns an **ID token** (a JWT) with user identity: name, email, etc.
      * This is what powers "Sign in with Google"

---

# Authorization

* *What can you do?*
* Giving specific permissions to specific users
* **RBAC — Role-Based Access Control**
   * Each user gets a role (e.g. `admin`, `editor`, `viewer`)
   * Each role has a set of permissions

   | Role | Read | Edit | Delete |
   |---|---|---|---|
   | viewer | ✅ | ❌ | ❌ |
   | editor | ✅ | ✅ | ❌ |
   | admin | ✅ | ✅ | ✅ |

   * Server identifies the role on login (stored in session or JWT)
   * On every next request, server checks the role and allows / denies the action
   * Denied → `403 Forbidden`
* Always check permissions on the **server** — hiding a button on the frontend is not security

---

# Security Practices

* **Send generic error messages**
   * Say "Invalid username or password", not "User not found" or "Wrong password"
   * Specific messages tell an attacker which usernames exist and what to try next

* **Timing attacks**
   * A typical login does these steps
      * Find user
      * Check if user is locked (lock users after too many invalid attempts)
      * Compare given password with the stored hash
   * Password hashing (bcrypt/argon2) is deliberately slow, e.g. ~100 ms
   * If the username doesn't exist, we return early and skip the hash → response comes in ~5 ms
   * Attacker measures the time
      * Fast reply (~5 ms) → username doesn't exist
      * Slow reply (~100 ms) → username exists
   * So the attacker learns at which step the request failed, even with a generic message
   * **Fix**
      * Use constant-time execution — run a dummy hash comparison even when the user isn't found, so every path takes the same time
      * Or add an artificial delay so all responses take similar time
      * Use constant-time comparison functions for secrets/tokens

* **Other basics**
   * Never store plain passwords — store only a hash (bcrypt / argon2) with salt
   * Rate-limit login attempts (e.g. 5 per minute per IP / user) → `429 Too Many Requests`
   * Always use HTTPS so tokens and cookies can't be sniffed
