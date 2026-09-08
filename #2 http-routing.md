# Understanding HTTP Routing: Mapping Requests to Server Logic

This video explains **routing** — the mechanism servers use to map incoming requests to the correct piece of backend logic — building directly on the HTTP methods and semantics covered in the previous video in the series.

## Key Takeaways

- HTTP methods express the **"what"** of a request (the intent), while routing expresses the **"where"** (the resource) — together, the method and route form a unique key that servers use to map a request to a specific handler.
- Routes come in several flavors — **static**, **dynamic (path parameters)**, **query parameters**, and **nested routes** — each suited to expressing a different kind of semantic meaning about the resource being requested.
- **Route versioning** and **catch-all routes** are practical patterns that let APIs evolve safely over time and handle unmatched requests gracefully instead of failing silently.

## 1. What is Routing?

**Routing** is the process of mapping a request's **URL path** (and its HTTP method) to a specific piece of server-side logic — a handler — that knows how to fulfill that request. If HTTP methods describe *what* a client wants to do (fetch, create, update, delete), routing describes *where* that action should be performed — which resource, endpoint, or piece of data the action applies to.

For example, a `GET` request to `/users` tells the server: "I want to fetch data, and specifically, I want the users resource." The server takes both pieces of information — the method and the path — and maps them to a handler that executes the relevant business logic (database queries, data transformations, etc.) and returns a response. In short: **routing maps URL parameters to server-side logic.**

## 2. Technical Flow (The "Hops")

The lifecycle of a routed request, based on the video's live demos in Burp Suite, works as follows:

1. **Client constructs a request**: It picks an HTTP method (GET, POST, etc.) and a route (e.g., `/api/books`).
2. **Server receives the request** and extracts two key pieces of information: the **method** and the **path**.
3. **Method + path form a unique key**: The server concatenates these two values internally to identify a single, unambiguous routing target — this is why the same path (`/api/books`) can safely support both a `GET` and a `POST` without conflict, since the method differentiates them.
4. **Route matching**: The server compares the incoming path against its registered route patterns. For dynamic routes, it recognizes a placeholder (e.g., `:id`) and treats whatever value appears in that position as a **path parameter**, converting it to a string regardless of its original format (even if it looks like a number).
5. **Handler execution**: Once matched, the server invokes the corresponding handler, which performs the necessary business logic — reading a path parameter, applying a query filter, executing a database call, etc.
6. **Fallback handling**: If no registered route matches the request, it falls through to a **catch-all handler**, typically registered last, which returns a friendly "not found" message instead of a blank or null response.
7. **Response returned**: The server sends back the appropriate data (or error) to the client.

## 3. Why Do We Need Routing?

- **Clear separation of intent and target**: Without routing, a server would have no structured way to know *which* resource an action applies to — methods alone aren't enough.
- **Human-readable, semantic API design**: Well-structured routes (like `/users/123/posts/456`) let both developers and systems understand at a glance what data is being requested, which is central to REST API design philosophy.
- **Collision avoidance**: Combining method and path into a single matching key allows the same path to serve multiple distinct operations (GET vs. POST on `/api/books`) without ambiguity.
- **Safe API evolution**: Versioned routes let a server introduce breaking changes (new response formats, new fields) without immediately breaking every existing client.
- **Graceful failure handling**: A catch-all route ensures that unmatched requests get a meaningful, user-friendly response instead of an unhandled error or empty payload.

## 4. Key Comparisons: Path Parameters vs. Query Parameters

Both path parameters and query parameters let a client send dynamic values to the server, but they serve different purposes and aren't interchangeable in practice.

- **Path parameters** (e.g., `/users/:id`) are part of the route itself and carry **semantic meaning** — they identify *which specific resource* is being acted on (e.g., "the user with ID 123").
- **Query parameters** (e.g., `/search?query=some+value`) are appended after a `?` as key-value pairs and are used to pass **metadata or modifiers** about a request — filters, sort order, pagination values — especially useful because `GET` requests don't have a body to carry this kind of data.

### Why Can't We Just Use Path Parameters for Everything?

1. **Semantic mismatch**: Path parameters are meant to identify a specific resource in a hierarchy (`/users/123`). Cramming arbitrary, non-identifying values (like a free-text search term) into the path — e.g., `/search/some-value` — is technically possible but breaks the intended meaning of REST-style URLs and becomes hard to maintain as more parameters are added.
2. **No support for multiple, optional values**: A search endpoint might need several independent modifiers at once — a search string, a sort field, a sort direction, a page number. Path parameters aren't designed for this; query parameters, as key-value pairs, handle this naturally (e.g., `?query=value&sort=asc&page=2`).
3. **GET requests lack a body**: POST and PUT requests can carry arbitrary data in the request body, but GET requests conventionally don't. Query parameters exist specifically to give GET requests a structured way to carry extra data without a body.

## 5. Deep Dive: Route Types and Patterns

### Static Routes

A **static route** (e.g., `/api/books`) contains no variable segments — it's a fixed string that always maps to the same handler and (logically) represents the same resource collection, regardless of when or how often it's called.

### Dynamic Routes and Path Parameters

A **dynamic route** includes a variable segment, conventionally written with a colon prefix (e.g., `/api/users/:id`) — a convention the video notes is common across virtually every backend language and framework (Node.js, Java, Python, Go, Rust, etc.), even though exact syntax varies. When a request like `/api/users/123` arrives, the server matches the static portions (`/api/users`) and extracts `123` into the `id` slot as a **path parameter** (also called a **route parameter**). This design gives REST APIs their readability: the request clearly reads as "get me data about the user with ID 123."

### Query Parameters

**Query parameters** are key-value pairs appended to a URL after a `?` (e.g., `/api/search?query=some+value`), separated by `&` when there are multiple. They're most commonly used with GET requests to pass filtering, sorting, or pagination data. The video illustrates this with a **pagination** example: a `/api/books` endpoint might return a `data` array of books alongside metadata like `total`, `currentPage`, and `totalPages`. To request a different page, the client simply appends `?page=2` to the same route rather than needing a distinct endpoint for each page.

### Nested Routes

**Nested routes** aren't a distinct routing mechanism so much as a common *pattern* for expressing hierarchical, semantic relationships between resources. For example:
- `/api/users` → all users
- `/api/users/123` → a specific user
- `/api/users/123/posts` → all posts belonging to user 123
- `/api/users/123/posts/456` → a specific post (456) belonging to a specific user (123)

Each additional segment narrows the semantic scope of the request, and each level can independently map to its own handler. This pattern shows up constantly in APIs of even moderate complexity.

### Route Versioning and Deprecation

**Route versioning** — embedding a version identifier in the route (e.g., `/api/v1/products` vs. `/api/v2/products`) — lets a server change a response format or behavior for new clients (e.g., a new mobile app) without breaking existing clients still relying on the old format. In the video's demo, `/api/v1/products` returned objects with `id`, `name`, `price`, while `/api/v2/products` returned `id`, `title`, `price` — a small but realistic example of a breaking change. Versioning gives teams a **migration window**: v1 can be marked deprecated once v2 ships, giving client-side engineers time to migrate before v1 is eventually retired.

*Fact-check note*: The video demonstrates **URI path versioning** (`/v1/`, `/v2/`), which remains one of the two dominant real-world approaches alongside **header-based versioning** (e.g., a custom `X-API-Version` header, as used by GitHub and Stripe, or an `Accept` media-type header). URI versioning is praised for being explicit and easy to discover, while header versioning is often considered more strictly "RESTful" since it keeps the URI focused purely on the resource. Both remain valid, widely used strategies in 2026 — the choice is generally a team/API-design preference rather than one being objectively deprecated.

### Catch-All Routes

A **catch-all route** (often written as `/*` or a wildcard) is registered last, after all specific routes have been defined. Any request that doesn't match a more specific route falls through to this handler, which returns a clear, user-friendly "route not found" message rather than a default null or empty response — improving the debugging experience for API consumers.

## 6. Interview Questions & Key Concepts

### Fundamentals

1. **What is the difference between a static route and a dynamic route?**
   *Talking Point: A static route is a fixed path with no variable segments and always maps to the same handler (e.g., `/api/books`). A dynamic route includes a placeholder segment (e.g., `/api/users/:id`) that captures a variable value from the actual request path, which the handler can extract and use — commonly to identify a specific resource by ID.*

2. **What's the difference between a path parameter and a query parameter, and when would you use each?**
   *Talking Point: Path parameters are part of the URL structure and identify a specific resource (e.g., `/users/123`), carrying semantic meaning about hierarchy. Query parameters (e.g., `?sort=asc&page=2`) are used for optional, non-identifying metadata like filters, sorting, or pagination — especially useful for GET requests, which don't have a request body to carry that information otherwise.*

3. **How does a server disambiguate between a GET and a POST request to the same URL path?**
   *Talking Point: Servers combine the HTTP method and the route path together to form a unique routing key. Two requests to `/api/books` — one GET, one POST — are treated as entirely separate routes internally because the method is part of what's matched, not just the path.*

4. **What happens when a client requests a route that doesn't exist on the server?**
   *Talking Point: Well-designed APIs register a catch-all (wildcard) route as a final fallback after all specific routes. Unmatched requests are routed here and typically return a 404 status with a clear, user-friendly error message, rather than an unhandled exception or a null response.*

### Architecture & Security

5. **Why is API versioning important, and what are the common strategies for implementing it?**
   *Talking Point: Versioning lets an API introduce breaking changes (new fields, restructured responses) without immediately breaking existing clients. The two most common strategies are URI path versioning (`/v1/resource`, `/v2/resource`) and header-based versioning (a custom header like `X-API-Version` or an `Accept` media type). URI versioning is simpler and more discoverable; header versioning keeps URLs resource-focused and is considered more strictly RESTful. Real-world APIs like GitHub and Stripe use header versioning, while many public APIs still favor URI versioning for its simplicity.*

6. **How would you design a deprecation strategy for an old API version?**
   *Talking Point: A good deprecation strategy involves releasing the new version alongside the old one, clearly communicating a deprecation timeline to consumers (often via documentation or deprecation headers like `Sunset` or `Deprecation`), monitoring usage of the old version, and only removing it once traffic has migrated or a defined support window has elapsed — avoiding an abrupt breaking change for clients who haven't migrated yet.*

7. **What security considerations come into play when accepting path or query parameters directly into business logic?**
   *Talking Point: Any parameter pulled from the path or query string is user-controlled input and must be validated and sanitized before use — failing to do so can expose the application to injection attacks (e.g., SQL injection if an ID is used unsanitized in a database query) or unintended access if authorization isn't checked against the resource identified by that parameter (e.g., insecure direct object reference).*

8. **How do nested routes relate to resource-based authorization?**
   *Talking Point: Nested routes like `/users/:userId/posts/:postId` naturally express ownership relationships, which makes it easier to enforce authorization — the server can check that the post identified by `postId` actually belongs to the user identified by `userId` before returning data, preventing one user from accessing another's nested resources by simply changing an ID.*

### Business Logic

9. **How would you design pagination for a large collection endpoint?**
   *Talking Point: Pagination typically uses query parameters (e.g., `?page=2&limit=20`) so the client can request data in manageable chunks. The response usually includes metadata such as total item count, current page, and total pages, letting the client build "next/previous" navigation and calculate whether more pages exist without an extra request.*

10. **Why might you choose nested routes over flat routes with query parameters for related resources (e.g., a user's posts)?**
    *Talking Point: Nested routes (`/users/:id/posts`) express a clear ownership or containment relationship semantically, making the API more intuitive and self-documenting. Flat routes with a filter (`/posts?userId=123`) can achieve a similar result and are sometimes preferred for flexibility (e.g., filtering posts by multiple optional criteria at once), so the choice often comes down to how strongly the relationship should be expressed in the URL versus how flexible the querying needs to be.*

11. **How would you extend an endpoint to support filtering and sorting without breaking existing clients?**
    *Talking Point: New optional query parameters (e.g., `?sortBy=price&order=desc`) can typically be added without impacting clients that don't send them, since well-designed handlers apply sensible defaults when a parameter is absent. This is generally safer than a breaking change and often doesn't require a new API version at all.*

12. **What's the tradeoff between versioning an entire API globally versus versioning individual endpoints?**
    *Talking Point: Global versioning (e.g., every route under `/v2/`) is simpler to reason about and communicate but forces a full migration even for unaffected endpoints. Per-endpoint versioning is more granular and can reduce migration overhead, but adds complexity in tracking which version of which endpoint a given client is using — the right choice depends on how frequently breaking changes occur and how tightly coupled the API's resources are.*

---

*Approximate word count: 2,150 words*
