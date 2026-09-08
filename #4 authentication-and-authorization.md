# Understanding Authentication and Authorization: Identity and Permission in Backend Systems

This video traces authentication and authorization from their historical roots to today's modern protocols, covering sessions, JWTs, cookies, stateful vs. stateless authentication, API keys, OAuth 2.0, OpenID Connect, and role-based authorization.

## Key Takeaways

- **Authentication** answers "who are you?" while **authorization** answers "what can you do?" — every mechanism discussed in this video (sessions, JWTs, OAuth, RBAC) exists to answer one of these two questions.
- Authentication has evolved through distinct trust models over time — from **something you have** (wax seals) to **something you know** (passwords) to **multi-factor combinations** (something you know + have + are) — and each stage introduced new vulnerabilities that pushed the next evolution forward.
- Modern systems typically combine **stateful authentication** (sessions, revocable, but harder to scale) with **stateless authentication** (JWTs, scalable, but hard to revoke) depending on the use case, while **OAuth 2.0** and **OpenID Connect** solve the separate problem of letting one platform securely access another's resources on a user's behalf.

## 1. What is Authentication and Authorization?

**Authentication** is the process of establishing identity — answering the question "who are you?" in a given context (a platform, an operating system, a device). **Authorization** is the process of determining permissions — answering "what can you do?" once your identity is known. These are related but distinct concerns: a user can be successfully authenticated (the system knows who they are) yet still be denied a particular action because they lack the required permissions.

## 2. Technical Flow (The "Hops")

### Stateful Authentication Flow

1. The client sends a username/email and password to the server.
2. The server validates the credentials.
3. If valid, the server generates a **session ID**, bundles it with the user's data, and stores this in a persistent store (commonly **Redis**, chosen for its fast read access compared to traditional databases).
4. The server sends the session ID back to the client inside an **HTTP-only cookie** (inaccessible to JavaScript).
5. On every subsequent request, the browser automatically attaches this cookie.
6. The server looks up the session ID in its persistent store to retrieve user data, checks expiry, and authenticates/authorizes the request accordingly.

### Stateless Authentication Flow (JWT-based)

1. The client sends credentials to the server on login.
2. The server validates them and, if correct, generates a **signed JWT** containing user information (ID, role, etc.), signed with a secret key the server holds.
3. The JWT is sent back to the client.
4. On subsequent requests, the client includes the JWT — conventionally in an `Authorization` header.
5. The server verifies the JWT's signature using its secret key and extracts the user's identity directly from the token payload — no database or cache lookup required.
6. If verification succeeds, the request proceeds; if not, the server returns an unauthorized/forbidden error.

### OAuth 2.0 + OpenID Connect Flow (Delegated Access with Identity)

1. A user clicks "Sign in with Google" (or similar) on a client platform (e.g., a note-taking app).
2. The client redirects the user to the **authorization server** (e.g., Google).
3. The user logs into the authorization server directly (not the client) and grants the requested permissions.
4. The authorization server sends back an **authorization code** (and, in OpenID Connect, an **ID token**) to the client.
5. The client exchanges the authorization code with the resource server for an **access token** (and an ID token, if not already received).
6. The client uses the access token to act on the user's behalf against the resource server (e.g., fetching notes from Google Keep), while the ID token (a JWT) provides the client with the user's authenticated identity.

## 3. Why Do We Need Authentication and Authorization?

- **Identity verification prevents impersonation**: Without authentication, any request could claim to be anyone, making meaningful access control impossible.
- **Granular permission control protects sensitive operations**: Not every authenticated user should be able to perform every action — authorization ensures capabilities like deleting data or accessing admin features are properly restricted.
- **Statelessness enables scale**: In distributed, multi-region, microservice architectures, stateful session lookups introduce latency and synchronization overhead; stateless tokens let any server independently verify a user without a shared lookup.
- **Delegation avoids password sharing**: Before OAuth, the only way for one platform to access another's resources was to share a password outright — granting total, unrevokable access. OAuth solves this with scoped, revocable tokens.
- **Secure error handling prevents information leakage**: Generic authentication error messages and consistent response timing prevent attackers from using the system's own feedback to refine their attacks.

## 4. Key Comparisons: Stateful vs. Stateless Authentication

|  | **Stateful (Sessions)** | **Stateless (JWT)** |
|---|---|---|
| **Storage** | Server-side (Redis/DB) session store | Self-contained token, no server storage |
| **Revocation** | Easy — delete the session | Hard — token valid until expiry unless blacklisted |
| **Scalability** | Requires synchronized session storage across servers | Naturally scalable across distributed servers |
| **Best for** | Web apps needing real-time session control | APIs, mobile apps, distributed/microservice systems |

### Why Can't We Just Use Stateless (JWT) Authentication for Everything?

1. **Token revocation is difficult**: Since a JWT is self-contained and stateless, the server has no built-in way to invalidate a specific token before it expires — short of rotating the entire signing secret, which would log out every user on the platform.
2. **No real-time visibility into active sessions**: A stateful system gives the server a live view of who is currently logged in and lets it forcibly log out a specific user; a pure JWT system has no equivalent mechanism without reintroducing server-side state (e.g., a blacklist).
3. **Security-sensitive compromise scenarios are harder to contain**: If a user's account is compromised, immediately cutting off access is straightforward with sessions (delete the session) but awkward with JWTs (requiring a blacklist lookup, which partially defeats the purpose of statelessness).

*Fact-check note*: The video's proposed "hybrid" fix — maintaining a blacklist of revoked JWTs in persistent storage — is a real, commonly used pattern in production systems, though as the video itself points out, it does reintroduce a storage lookup, partially undermining the pure statelessness JWTs are meant to provide. In practice, many teams address this by using **short-lived access tokens** paired with **long-lived, rotatable refresh tokens**, which limits the exposure window without requiring a lookup on every request.

## 5. Deep Dive: Historical Context, Core Components, and Modern Protocols

### A Brief History of Authentication

The video traces authentication's evolution through several eras:

- **Pre-industrial societies**: Identity was tied to personal recognition — a trusted community member (e.g., a village elder) could vouch for someone, and agreements were sealed with a handshake. This relied on **human contextual trust** and didn't scale beyond small communities.
- **Medieval period**: **Wax seals** emerged as one of the first cryptographic authentication tokens, relying on the principle of **"something you have."** Seals were vulnerable to forgery, marking some of the earliest recorded authentication bypass attacks.
- **Industrial Revolution / telegraph era**: Telegraph operators used pre-agreed static **passphrases**, shifting the underlying principle to **"something you know."**
- **Mainframe era (1961)**: Researchers at MIT's Project MAC introduced passwords for multi-user systems (**CTSS**). A now-famous incident — someone printing the plaintext password file — exposed the danger of storing passwords in plaintext and catalyzed the shift toward **hashing** (a one-way, fixed-length transformation of a password) for secure password storage.
- **1970s cryptographic research**: Whitfield Diffie and Martin Hellman's **Diffie-Hellman key exchange** introduced **asymmetric cryptography**, enabling two parties to establish a shared secret over an untrusted channel — the foundation of modern **PKI (Public Key Infrastructure)**. This era also produced **Kerberos**, a ticket-based system relying on a trusted third party, foreshadowing modern token-based authentication.
- **1990s**: As brute-force and dictionary attacks grew more sophisticated, simple username/password systems proved insufficient, leading to **Multi-Factor Authentication (MFA)** — combining "something you know" (passwords/PINs), "something you have" (smart cards, OTP generators), and "something you are" (biometrics).
- **21st century**: Cloud computing, mobile devices, and API-driven architectures drove demand for more advanced frameworks — **OAuth**, **JWTs**, **zero trust architecture**, and **passwordless authentication** (e.g., WebAuthn, which relies on public/private key pairs stored in hardware).
- **Emerging/future directions** mentioned: **decentralized identity** (often blockchain-based), **behavioral biometrics**, and **post-quantum cryptography** — the effort to design cryptographic algorithms resistant to the eventual computational power of quantum computers.

*Fact-check note*: This historical narrative is broadly accurate and well-established. On post-quantum cryptography specifically: NIST finalized its first set of **post-quantum cryptographic standards** (including ML-KEM and ML-DSA) in 2024, and adoption/migration planning has continued to progress into 2026, making this an active, ongoing industry effort rather than a purely speculative future concern.

### Three Core Components: Sessions, JWTs, and Cookies

- **Sessions**: A mechanism for giving the inherently stateless HTTP protocol "memory" of a user across requests. Early implementations stored session data in files on the server (poor scalability), later moving to databases (better persistence, still slower), and eventually to distributed in-memory stores like **Redis** or **Memcached** for speed and scalability across multiple servers.
- **JWTs (JSON Web Tokens)**: Formalized in 2015, JWTs are a stateless mechanism for transferring claims between parties. A JWT has three base64-encoded parts:
  - **Header**: metadata such as the signing algorithm.
  - **Payload**: the actual claims — conventionally including `sub` (subject/user ID) and `iat` (issued-at timestamp), plus optional custom fields like name and role.
  - **Signature**: a cryptographic signature (using a secret key) that lets the server verify the token hasn't been tampered with.

  JWTs offered three major advantages over pure session-based systems: **statelessness** (no server-side storage cost), **scalability** (any server holding the shared secret can independently verify a token — ideal for microservices), and **portability** (a compact, URL-safe string that can be passed via headers, cookies, or URLs). Their two major drawbacks are **difficult revocation** (no built-in way to invalidate a token before expiry) and **limited real-time visibility** into which tokens are still "active."

- **Cookies**: A browser-provided mechanism letting a server store a small piece of data in a client's browser, scoped so that only the originating server can read it. Once set, the browser automatically attaches the cookie to every subsequent request to that server, which is what makes cookies convenient for automatically carrying session IDs or tokens.

### Four Major Authentication Types

1. **Stateful authentication**: Session-based, backed by a persistent store; offers centralized control, real-time session visibility, and easy revocation, at the cost of scalability challenges in distributed systems.
2. **Stateless authentication**: JWT-based; offers scalability and no session-store dependency, at the cost of difficult token revocation.
3. **API key-based authentication**: Designed for **machine-to-machine communication** rather than human/UI-driven login — e.g., a developer generating an API key to programmatically call a service like OpenAI's API instead of interacting with its UI. Advantages include ease of generation and suitability for programmatic, non-interactive access with defined permissions and expiry.
4. **OAuth 2.0 / OpenID Connect**: Designed for **delegated access** and **third-party login** — letting one platform securely access another's resources (or verify identity) without ever handling the user's password.

**Recommended pattern**: Use stateful authentication for browser-based web apps (where real-time revocation matters), stateless authentication for APIs and distributed/mobile systems, API keys for server-to-server integrations, and OAuth/OIDC for third-party login and delegated resource access. The video also strongly recommends that, for production systems, teams use established auth providers (e.g., Auth0, Clerk) rather than building authentication from scratch, reserving custom implementations for learning purposes.

### The Evolution from OAuth 1.0 to OAuth 2.0 and OpenID Connect

**The delegation problem**: As platforms began needing access to each other's resources (e.g., a travel app scanning a user's Gmail for flight tickets), the initial "solution" — sharing passwords outright — was disastrous: it granted unlimited access with no way to scope or revoke it without changing the password everywhere.

**OAuth 1.0** (2007) solved this by introducing **token sharing** instead of password sharing, with four core roles: the **resource owner** (the user), the **client** (the app requesting access), the **resource server** (where the data lives), and the **authorization server** (which issues tokens). A token carries limited, specific permissions rather than full account access.

**OAuth 2.0** (~2010) addressed OAuth 1.0's complexity and its error-prone reliance on cryptographic request signing by introducing simpler **bearer tokens** and multiple **flows** tailored to different client types:
- **Authorization code flow**: for server-side apps.
- **Implicit flow**: originally for browser-based apps.
- **Client credentials flow**: for machine-to-machine communication.
- **Device code flow**: for input-constrained devices (e.g., smart TVs).

*Fact-check note*: The video correctly flags the implicit flow as discouraged for security reasons. This has since hardened into formal guidance: modern **OAuth 2.0 Security Best Current Practice** and the newer **OAuth 2.1** specification explicitly deprecate and remove both the **implicit grant** and the **Resource Owner Password Credentials (ROPC)** grant, mandating the **Authorization Code flow with PKCE (Proof Key for Code Exchange)** for all client types, including browser-based single-page apps — not just public clients as originally recommended. Teams still using implicit flow should be actively migrating to Authorization Code + PKCE.

Critically, **OAuth 2.0 solves authorization (delegated access) but not authentication (identity verification)**. This gap was filled by **OpenID Connect (OIDC)**, introduced around 2014, which layers an **ID token** (itself a JWT, containing identity claims like user ID, issuer, and issue time) on top of the OAuth 2.0 flow. This is the mechanism behind ubiquitous "Sign in with Google/Facebook/Discord" buttons — OIDC lets a platform confirm a user's identity through a trusted third party without implementing its own authentication system.

### Role-Based Access Control (RBAC)

Once a user is authenticated, **authorization** determines what they're permitted to do. **Role-Based Access Control (RBAC)** assigns users to roles (e.g., user, admin, moderator), each carrying a defined set of permissions on specific resources (e.g., a "user" role might have read-only access to certain data, while an "admin" role has read/write/delete access, including to restricted areas). In a typical flow, the server determines a user's role early in the request lifecycle (via the session or JWT) and passes that information along so downstream logic can enforce access — returning a **403 Forbidden** if a user attempts an action outside their role's permissions.

*Fact-check note*: RBAC remains a foundational and widely used authorization model. For more complex or fine-grained needs, many modern systems complement or replace RBAC with **Attribute-Based Access Control (ABAC)** or **Relationship-Based Access Control (ReBAC)**, which evaluate permissions based on contextual attributes or resource relationships rather than fixed roles alone — worth knowing as a natural next step beyond RBAC, though RBAC's role-and-permission model remains the standard starting point covered in the video.

### Security Practices: Generic Error Messages and Timing Attacks

- **Avoid specific authentication error messages**: Returning distinct messages like "user not found" versus "incorrect password" gives attackers a way to confirm which piece of a credential pair is valid, narrowing their attack surface for credential stuffing or brute-force attempts. The recommended practice is to always return a single, generic message (e.g., "authentication failed") regardless of which check actually failed.
- **Guard against timing attacks**: Because operations like password hashing take measurably longer than a simple "user not found" lookup, an attacker can sometimes infer whether a username exists based on how long the server takes to respond. Mitigations include using **constant-time comparison functions** for password hashes and, where needed, deliberately simulating a consistent artificial delay so that response times don't leak information about which validation step failed.

## 6. Interview Questions & Key Concepts

### Fundamentals

1. **What is the difference between authentication and authorization?**
   *Talking Point: Authentication verifies identity — confirming who a user is, typically via credentials, tokens, or biometrics. Authorization determines permissions — what an authenticated user is allowed to do. A user can be authenticated but still be denied a specific action if they lack the required authorization.*

2. **How does session-based (stateful) authentication work, and what are its trade-offs?**
   *Talking Point: The server creates a session ID on login, stores associated user data in a persistent store (often Redis for speed), and sends the session ID to the client via an HTTP-only cookie. Every subsequent request includes this cookie, letting the server look up the session. This offers strong control (easy revocation, real-time visibility) but requires synchronized storage across servers in distributed deployments, which introduces latency and operational complexity.*

3. **What is a JWT, and what are its three components?**
   *Talking Point: A JSON Web Token is a self-contained, stateless credential with three base64-encoded parts: a header (metadata like the signing algorithm), a payload (claims like user ID and role), and a signature (cryptographically verifying the token hasn't been tampered with, using a secret key held by the issuing server).*

4. **Why can't JWTs be easily revoked before they expire?**
   *Talking Point: Because JWTs are self-verifying via a signature check rather than a server-side lookup, there's no built-in mechanism to mark one as invalid early. Common mitigations include maintaining a token blacklist (reintroducing some server-side state), using short-lived access tokens with separately revocable refresh tokens, or rotating the signing secret (which invalidates all tokens, not just one).*

### Architecture & Security

5. **Walk through the OAuth 2.0 authorization code flow and explain why PKCE matters.**
   *Talking Point: The client redirects the user to an authorization server, the user authenticates and grants permissions, and the authorization server returns an authorization code, which the client exchanges (server-side) for an access token. PKCE adds a dynamically generated secret ("code verifier") that must match on both the initial request and the token exchange, preventing an intercepted authorization code from being redeemed by an attacker. Modern OAuth 2.0 best practice — formalized in OAuth 2.1 — mandates PKCE for all client types, including server-side apps, not just public/browser-based ones, and has fully deprecated the older implicit flow.*

6. **What's the difference between OAuth 2.0 and OpenID Connect?**
   *Talking Point: OAuth 2.0 solves delegated authorization — letting one application access another's resources on a user's behalf via scoped access tokens — but it doesn't verify identity. OpenID Connect extends OAuth 2.0 by adding an ID token (a JWT containing identity claims), enabling authentication use cases like "Sign in with Google," which OAuth 2.0 alone cannot provide.*

7. **Why should authentication error messages be generic rather than specific?**
   *Talking Point: Specific messages like "user not found" or "incorrect password" leak information that helps attackers refine credential-stuffing or brute-force attacks — confirming a username's existence, for instance, narrows their search space. Best practice is a single generic message (e.g., "authentication failed") regardless of which validation step actually failed, combined with rate limiting and account lockout policies.*

8. **What is a timing attack in the context of authentication, and how do you defend against it?**
   *Talking Point: A timing attack exploits measurable differences in server response time between different failure points (e.g., a fast rejection for a nonexistent username versus a slower rejection after hashing and comparing a password), letting an attacker infer which part of a credential pair is correct. Defenses include constant-time comparison functions for password hashes and, in some designs, deliberately equalizing response times with simulated delays.*

### Business Logic

9. **When would you choose stateful versus stateless authentication for a new system?**
   *Talking Point: Stateful (session-based) authentication suits browser-based web applications where real-time session control and easy revocation matter, such as most SaaS products. Stateless (JWT-based) authentication suits APIs, mobile apps, and distributed/microservice architectures where scalability and avoiding centralized session lookups are priorities. Many real systems use a hybrid: sessions for the primary web app, JWTs for API/mobile clients.*

10. **What is an API key, and how does it differ from a user authentication token?**
    *Talking Point: An API key is typically issued for machine-to-machine or programmatic access — for example, calling a third-party service's API from your own backend — rather than representing an individual human's login session. API keys are usually simpler to generate and manage, often scoped with specific permissions and rate limits, and don't require the interactive login/token-refresh workflows used for human users.*

11. **How would you design a role-based authorization system for a multi-tenant application?**
    *Talking Point: Define a set of roles (e.g., owner, admin, member, viewer) with clearly scoped permissions per resource type, and determine a user's role early in the request lifecycle (from their session or JWT) so downstream logic can consistently enforce access — returning a 403 Forbidden when a user attempts an action outside their role's permissions. For finer-grained needs beyond fixed roles (e.g., per-resource ownership checks), many systems layer attribute-based or relationship-based checks on top of a base RBAC model.*

12. **Why is it considered bad practice to store passwords in plaintext, and what's the standard alternative?**
    *Talking Point: Storing plaintext passwords means a single database breach exposes every user's actual password, which is especially dangerous given how often people reuse passwords across services. The standard practice is to hash passwords with a slow, salted, cryptographically secure hashing algorithm (such as bcrypt, scrypt, or Argon2) before storage, so that even if the database is compromised, recovering the original passwords is computationally infeasible.*

---

*Approximate word count: 3,400 words*
