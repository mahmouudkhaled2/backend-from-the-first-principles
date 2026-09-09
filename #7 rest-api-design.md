# REST API Design: From Historical Roots to a Consistent, Intuitive Interface

[REST API Design](https://www.youtube.com/watch?v=RG6q57DwV8Y&list=PLui3EUkuMTPgZcV0QhQrOcwMPcBCcd_Q1&index=11&pp=iAQB) This video traces REST API design back to its historical origins and then walks through, in extensive hands-on detail, how to design a consistent, intuitive REST interface — covering resource naming, HTTP methods, idempotency, pagination, sorting, filtering, and custom actions.

## Key Takeaways

- **REST (Representational State Transfer)** emerged from Roy Fielding's 1990s effort to solve the early web's scalability crisis, and its name literally describes its three core ideas: resources have **representations** (formats), resources have **state**, and that state is **transferred** between client and server.
- **Consistency is the single most important characteristic of good API design** — sticking to one naming convention, one payload structure, and one set of behaviors across every endpoint eliminates guesswork for whoever integrates your API, whether or not you strictly follow the "official" REST standard.
- **Idempotency** — GET, PUT, and DELETE are idempotent, POST is not — directly determines which HTTP method fits a given operation, and understanding *why* each method has that property clarifies method selection far better than memorizing a lookup table.

## 1. What is REST API Design?

**REST (Representational State Transfer)** API design is the practice of structuring an API's resources, routes, HTTP methods, and payloads according to a widely adopted architectural style originally described by Roy Fielding in his year-2000 PhD dissertation. The goal of following this standard isn't to invent new rules, but to extract a consistent, predictable set of conventions from decades of established practice, so that backend engineers can focus on business logic instead of re-litigating basic design questions (should this route be singular or plural? should this be PATCH or PUT?) on every project.

## 2. Technical Flow (The "Hops")

The video presents API design as a structured workflow that happens *before* any code is written:

1. **Start from UI/UX designs**: Begin with wireframes (e.g., Figma) or direct conversations with product/client stakeholders to understand how end users will interact with the platform's data.
2. **Identify resources (nouns)**: From those requirements, extract the core nouns — the entities the platform manages (e.g., for a project management tool: organizations, projects, tasks, users, tags).
3. **Design the database schema**: Translate those resources into database tables (covered in a separate video in the series, only briefly touched on here).
4. **Identify actions per resource**: For each resource, determine what operations clients need — typically the four CRUD operations (Create, Read, Update, Delete), split into five endpoint types: create, list (get-all), get-single, update, delete.
5. **Design the route and method for each action**: Map each action to a URL path and HTTP method following REST conventions (e.g., `GET /organizations` for list, `POST /organizations` for create).
6. **Identify custom (non-CRUD) actions**: For operations that don't map cleanly to create/read/update/delete (e.g., "archive an organization," "clone a project"), design a dedicated action-based endpoint.
7. **Test and iterate the interface using an API client**: Use a tool like Insomnia or Postman (or an interactive spec tool like Swagger/OpenAPI) to validate the design end-to-end before writing any business logic or server code.
8. **Only then move to implementation**: Actual programming — the business logic, database queries, and framework-specific code — comes after the interface design is settled.

## 3. Why Do We Need REST API Design Standards?

- **Eliminates guesswork for API consumers**: A consistent, standards-following API means any engineer integrating it can predict its behavior — which method to call, what payload shape to expect, what status code signals success — without reading source code or trial-and-error testing.
- **Reduces bugs and support burden**: Following an established standard significantly cuts down on integration errors, confused sync-up calls, and repeated clarifying questions between API producers and consumers.
- **Enables independent evolution of client and server**: This traces directly back to REST's original client-server constraint — separating concerns lets frontend and backend teams iterate independently.
- **Supports scalability**: REST's statelessness constraint (each request is fully self-contained) means any server in a load-balanced pool can handle any request, which is essential for horizontally scaling a backend.
- **Improves caching and performance**: The REST caching constraint (responses explicitly marked cacheable or not) reduces server load and improves perceived client performance.
- **Professional credibility and maintainability**: A consistent, intuitive API signals engineering discipline and makes long-term maintenance dramatically easier than an ad hoc, inconsistent one.

## 4. Key Comparisons: PATCH vs. PUT for Updates

Both PATCH and PUT are used to modify an existing resource, but they express different semantic intent and have different practical implications.

- **PUT** is meant to **completely replace** a resource's representation — the client must send the entire resource (every field), and the server replaces the stored version wholesale.
- **PATCH** is meant to **partially update** a resource — the client sends only the fields that need to change, and the server merges those into the existing resource.

### Why Can't We Just Use PUT for Every Update?

1. **Modern APIs are JSON-heavy and field-partial by nature**: In today's single-page-application-driven world, clients typically want to change one or two fields (e.g., just a status field) rather than resubmitting an entire object — PATCH's semantics match this reality far better than PUT's full-replacement model, which was more suited to the multi-page application era.
2. **PUT requires the client to have and resend the complete, current resource state**: This is often impractical or unnecessary overhead when only a small part of a resource needs to change, and risks accidentally overwriting fields the client didn't intend to touch if its copy of the resource is stale.
3. **Semantic clarity for API consumers**: Using PATCH for partial updates signals intent clearly to anyone integrating the API; using PUT for a partial update is technically workable (many teams use PUT and PATCH interchangeably without catastrophic consequences) but creates a mismatch between the method's documented meaning and its actual behavior, which can cause confusion for engineers who assume the API follows standard semantics.

*Fact-check note*: This aligns with current, widely followed REST guidance — using PATCH as the default for updates in modern JSON APIs remains standard practice in 2026, with PUT reserved for genuine full-replacement scenarios.

## 5. Deep Dive: Historical Origins, Idempotency, and Building a Full API Interface

### The Historical Origins of REST

The video traces REST's origins through two key figures:

- **Tim Berners-Lee** (1990) created the foundational technologies of the World Wide Web within about a year: the **URI**, the **HTTP protocol**, **HTML**, the first web server, the first web browser, and the first WYSIWYG HTML editor.
- As the web's user base grew exponentially, it began heading toward a **scalability breakdown** — the original design hadn't accounted for that scale.
- **Roy Fielding** (co-founder of the Apache HTTP Server project), around 1993, proposed six architectural constraints to solve this scalability problem:
  1. **Client-server**: separating UI/UX concerns (client) from data storage and business logic (server), allowing each to evolve independently.
  2. **Uniform interface**: a standardized way for components to communicate, itself composed of four sub-constraints (resource identification, resource manipulation through representations, self-descriptive messages, and hypermedia as the engine of application state — HATEOAS).
  3. **Layered system**: an architecture of hierarchical layers where each layer only interacts with the layer immediately below it, enabling intermediate components like load balancers and proxies without disrupting core functionality.
  4. **Cacheable**: responses must be explicitly labeled as cacheable or non-cacheable, reducing server load and improving client-perceived performance.
  5. **Stateless**: every request must carry all the information needed to process it — the server retains no memory of prior requests, which is critical for horizontal scalability across load-balanced servers.
  6. **Code on demand** (optional): servers can extend client functionality by sending executable code (e.g., JavaScript) to the client.
- Fielding and Berners-Lee later collaborated on standardizing **HTTP 1.1**. In **2000**, Fielding formally named and described this architectural style **REST (Representational State Transfer)** in his PhD dissertation — the foundational document behind everything we now call REST APIs.

*Fact-check note*: This historical account is accurate and matches the well-documented record of Fielding's dissertation, "Architectural Styles and the Design of Network-based Software Architectures" (University of California, Irvine, 2000), which remains the canonical primary source for REST.

### What "REST" Actually Means

Breaking down the name itself:

- **Representational**: The same underlying resource (e.g., a "user") can be represented in different formats depending on context — JSON for server-to-server or API-client communication, HTML for browser-rendered content, or XML in some systems.
- **State**: Refers to a resource's current condition or attributes (e.g., a shopping cart's items, quantities, and total price) — a snapshot that gets transferred between client and server.
- **Transfer**: The movement of a resource's representation between client and server, conducted via HTTP methods (GET, POST, PUT, PATCH, DELETE, etc.).

### URL Structure for APIs

A typical, well-structured API URL follows this pattern: **scheme** (`https`) → **subdomain** (commonly `api.`) → **domain** (`example.com`) → **version** (`/v1`) → **resource path** (`/books`). Key rules covered:

- **Resource names in the path must always be plural** — even when fetching a single item (e.g., `GET /books/{id}`, not `GET /book/{id}`), because the path segment represents the resource *collection*, and the specific item is identified via the ID that follows it.
- **No spaces or underscores in URLs.** When a human-readable identifier (a "slug") is needed — e.g., representing a book titled "Harry Potter" — it should be lowercased and spaces replaced with hyphens (`harry-potter`), since URLs travel across different environments (servers, clients, operating systems) where case and character handling can be inconsistent.
- **Forward slashes (`/`) represent hierarchical relationships** between resources — e.g., `/organizations/{id}/projects` expresses "the projects belonging to this specific organization."

### Idempotency and HTTP Method Selection

**Idempotency** means that performing the same action multiple times produces the same effect as performing it once — specifically, in the REST context, it means the *side effect on the server* doesn't change no matter how many times an identical request is repeated.

- **GET is idempotent**: fetching data any number of times causes no server-side side effects.
- **PUT and PATCH are idempotent**: replacing or updating a resource with the same payload repeatedly leaves the resource in the same final state after the first call as after any subsequent identical call.
- **DELETE is idempotent**: deleting an already-deleted resource causes no *further* side effect — the second call simply returns a 404 because the resource no longer exists, but no additional state change occurs.
- **POST is the only major non-idempotent method**: each identical POST call (e.g., creating a new book) typically creates a *new* distinct resource with a new database-generated ID, meaning the cumulative side effect grows with each call.

Because POST is the odd one out — and because REST doesn't have a dedicated method for arbitrary, non-CRUD "actions" (e.g., "send an email," "archive an organization," "clone a project") — the spec treats **POST as open-ended**: whenever an operation doesn't cleanly map to create/read/update/delete, POST is the conventional choice for that **custom action**.

### Designing a Complete API: The Demo Workflow

Using a project-management-platform example (organizations, projects, tasks) built with Insomnia, the video walks through designing a full CRUD interface for each resource:

- **List (GET `/organizations`)**: Returns a **paginated** response — not the entire dataset — because serializing and transferring very large datasets is resource-intensive and introduces perceptible client-side delay, while users typically only view a small slice of data at once anyway.
- **Create (POST `/organizations`)**: Accepts only client-provided fields (excluding server-managed fields like ID, `createdAt`, `updatedAt`); returns **201 Created** with the newly created entity.
- **Get single (GET `/organizations/{id}`)**: Returns **200 OK** with the resource, or **404 Not Found** if the ID doesn't exist.
- **Update (PATCH `/organizations/{id}`)**: Accepts partial fields; returns **200 OK** with the updated entity.
- **Delete (DELETE `/organizations/{id}`)**: Returns **204 No Content** with an empty body on success.
- **Custom action (POST `/organizations/{id}/archive`)**: For operations with broader side effects than a simple field update (e.g., archiving an organization might cascade to deleting associated projects, notifying users, etc.), a dedicated action endpoint is used rather than overloading PATCH — even though the endpoint uses POST, the response code depends on what actually happens server-side (the video's archive example still returns 200, not 201, since no new resource is created).

#### Pagination

Pagination limits how much data is returned per request. The video's example response structure includes:
- `data`: the current page's slice of results.
- `total`: the total count of matching resources across all pages (independent of the current page size).
- `page`: which page/portion this response represents.
- `totalPages`: how many pages exist in total (useful for the client to know when to stop requesting more, e.g., in infinite scroll).

Clients control pagination via **`limit`** (how many items per page) and **`page`** (which page to fetch) query parameters — both of which the server should default sensibly (e.g., `page=1`, `limit=10` or `20`) if the client omits them.

*Fact-check note*: The video demonstrates classic **offset/page-based pagination** (`page` + `limit`), which remains common and is perfectly reasonable for many applications, especially internal or moderately sized datasets. For very large or frequently changing datasets, current best practice increasingly favors **cursor-based pagination** — where the client passes an opaque cursor pointing to the last-seen item rather than a page number — since offset-based pagination can skip or duplicate records when data is inserted or deleted between requests, and its performance degrades on large tables as the database must scan past all skipped rows. Both approaches remain valid depending on data volume, but this is a worthwhile refinement to know when scaling beyond the video's example.

#### Sorting

List APIs should support **`sortBy`** (which field to sort by) and **`sortOrder`** (ascending or descending) query parameters — and critically, should apply **sensible defaults** even when the client sends neither (e.g., sorting by `createdAt` in descending order by default), since an unsorted database query can return results in an unpredictable, inconsistent order across identical requests.

#### Filtering

List APIs can support filtering via query parameters matching resource fields directly (e.g., `?status=archived`), letting clients narrow results without needing a separate endpoint per filter combination.

#### Consistency Across Resources and Fields

A recurring, heavily emphasized point: **once a naming or structural pattern is established for one resource, it must be followed identically for every other resource in the API.** If the `organizations` endpoint uses a `description` field, the `projects` endpoint must also use `description` — not an abbreviation like `desc`. Consumers naturally form assumptions based on the first endpoints they integrate, and breaking that pattern elsewhere forces them into unnecessary guesswork, documentation lookups, or trial-and-error debugging. This applies to route structure, payload field names (which should consistently use **camelCase** for JSON, per common JSON convention), default behaviors, and response shapes.

#### Empty Results: 404 vs. Empty 200

An important distinction: a **list** endpoint returning no matching results (e.g., a filter that matches nothing) should still return **200 OK** with an empty array — never a 404 — because the client is requesting a *collection*, not a specific entity. A **404** is reserved for requests targeting a *specific* resource by ID that doesn't exist (e.g., `GET /organizations/{id}` for a deleted organization).

### Additional Best Practices Highlighted

- **Maintain interactive API documentation** (e.g., Swagger/OpenAPI) from the start of a project — both as a testing tool during development and as living documentation for API consumers.
- **Provide sane defaults wherever reasonable** — not just for pagination/sorting, but also for fields like a resource's initial status (e.g., defaulting a newly created organization's status to "active" if the client doesn't specify one), minimizing the required payload to only what's truly necessary.
- **Avoid abbreviations in field names** — the people integrating an API don't share the same context as its designer, so field names should be fully spelled out and self-explanatory (`description`, not `desc`).
- **Design the interface before writing any code** — using a wireframe/requirements review, then an API client like Insomnia or Postman (or Swagger) to iterate on the interface itself, entirely separate from implementation details or programming language choice.

## 6. Interview Questions & Key Concepts

### Fundamentals

1. **What does REST actually stand for, and what does each part of the name mean?**
   *Talking Point: REST stands for Representational State Transfer. "Representational" refers to a resource being expressible in different formats (JSON, XML, HTML) depending on context. "State" refers to a resource's current attributes/condition. "Transfer" refers to the movement of that state's representation between client and server via HTTP. Together, these describe an architecture where resources are represented in various formats, their state is transferred between client and server, and the whole system follows constraints designed for scalability.*

2. **Why should resource names in a URL path always be plural, even for a single-item endpoint?**
   *Talking Point: The path segment represents the resource collection as a whole (e.g., "books"), and a specific item within that collection is identified by an additional path segment (its ID), such as /books/123. Using the plural form consistently — for both the list endpoint and the single-item endpoint — keeps the API's naming convention predictable and avoids inconsistency across an API's routes.*

3. **What is idempotency, and which HTTP methods are idempotent?**
   *Talking Point: An idempotent operation produces the same server-side effect no matter how many times it's repeated with the same input. GET, PUT, and DELETE are idempotent — repeating them doesn't compound any change. POST is not idempotent, since repeated identical calls typically create additional new resources each time, compounding the effect with every call.*

4. **When should you use PATCH versus PUT for an update operation?**
   *Talking Point: PATCH is used for partial updates, where the client sends only the fields that need to change. PUT is used for a full replacement, where the client sends the entire resource representation to overwrite the existing one. Modern JSON-based APIs predominantly use PATCH by default, since clients typically only need to modify a subset of fields rather than resubmit an entire resource.*

### Architecture & Security

5. **Why is statelessness one of REST's core constraints, and how does it support scalability?**
   *Talking Point: Statelessness requires every request to carry all the information the server needs to process it, with no reliance on stored context from prior requests. This allows any server in a horizontally scaled, load-balanced pool to handle any incoming request without needing shared session state, which is essential for scaling a system across many servers and regions.*

6. **How should an API handle a custom action that doesn't map to a standard CRUD operation (e.g., "archive an organization" or "clone a project")?**
   *Talking Point: Since REST's standard methods (GET, POST, PUT, PATCH, DELETE) map to create/read/update/delete semantics, an operation with broader or different side effects — such as cascading deletions, notifications, or complex business logic beyond a simple field change — doesn't fit cleanly into any of them. The convention is to expose it as a dedicated action-style endpoint (e.g., POST /organizations/{id}/archive) using POST, since POST is treated as the open-ended method in the REST specification for anything that isn't a standard CRUD operation.*

7. **What status code should a DELETE request return, and why?**
   *Talking Point: A successful DELETE typically returns 204 No Content, since the resource has been removed and there's no meaningful content to return in the response body. This differs from a successful GET or PATCH, which returns 200 OK along with the relevant resource data.*

8. **How should an API differentiate between "no matching data" and "resource not found"?**
   *Talking Point: A list endpoint that returns no matching results (e.g., due to a filter) should still return 200 OK with an empty array, since the client requested a collection, not a specific entity — there's nothing "not found" about an empty valid result set. A 404 Not Found is reserved for requests targeting a specific resource by identifier that doesn't exist, such as fetching a single deleted or nonexistent record by ID.*

### Business Logic

9. **How would you design pagination, sorting, and filtering for a list endpoint that needs to scale to a large dataset?**
   *Talking Point: Support limit and page (or a cursor-based equivalent) query parameters with sensible server-side defaults if omitted, along with sortBy/sortOrder and field-based filter parameters. For very large or frequently changing datasets, cursor-based pagination is generally preferred over offset/page-based pagination, since offset pagination can skip or duplicate records when data changes between requests and its performance degrades on large tables — though offset-based pagination, as shown in many introductory examples, remains adequate for smaller or more static datasets.*

10. **Why is consistency across an API's resources considered one of the most important qualities of good API design?**
    *Talking Point: Once a consumer integrates one endpoint and observes its conventions (field names, route structure, default behaviors), they naturally assume every other endpoint in the same API will follow the same pattern. Breaking that consistency — using different field names, casing, or structures for conceptually similar data across resources — forces unnecessary guesswork, documentation lookups, and debugging effort for anyone integrating the API, regardless of whether the API strictly follows the official REST standard.*

11. **What are sensible default values a well-designed API should apply, and why does this matter?**
    *Talking Point: Defaults should be applied wherever a reasonable assumption can be made without requiring explicit client input — e.g., defaulting pagination to page 1 with a set limit, defaulting sort order to newest-first, or defaulting a newly created resource's status field to a sensible starting value. This reduces the required payload to only what's truly necessary and prevents unnecessary validation errors for fields that have an obvious sensible default.*

12. **How would you decide whether an operation should be its own custom action endpoint versus just an update (PATCH) to a status field?**
    *Talking Point: If the operation's effect is limited to changing that single field's value with no other consequences, a PATCH to update the field is appropriate. If the operation has broader side effects — cascading changes to related resources, triggering notifications, or other business logic beyond the field change itself — it should be modeled as a distinct custom action endpoint, since conflating a complex operation with a simple field update misrepresents what the API call actually does and can obscure important side effects from API consumers.*

---

*Approximate word count: 3,350 words*
