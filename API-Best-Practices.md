# API Design Best Practices

Designing high-quality APIs is both an art and a science. A well-designed API is **intuitive**, **performant**, **secure**, **evolvable**, and **developer-friendly**. These best practices are battle-tested across large-scale production systems and follow REST principles while incorporating modern considerations for scalability and security.

## 1. Follow RESTful Principles (with Pragmatism)

- Use **resources** as the primary abstraction.
- Leverage **HTTP methods** as verbs on those resources.
- Keep URIs **predictable** and **hierarchical**.
- Make APIs **stateless** (each request contains all necessary context).
- Prefer **hypermedia** (HATEOAS) only when it adds real value—otherwise keep it simple.

**Rule of thumb**: If it feels like a natural URL a developer would guess, you’re doing it right.

## 2. Resource Naming: Nouns, Not Verbs

- **Use plural nouns** for resource collections.
- **Never use verbs** in the URL path (verbs belong in the HTTP method).

| ✅ Good | ❌ Bad |
|---------|--------|
| `GET /users` | `GET /getUsers` |
| `POST /orders` | `POST /createOrder` |
| `PATCH /users/123/preferences` | `POST /updateUserPreferences` |

- Use **nested resources** for relationships: `GET /users/123/orders`
- Use **kebab-case** or **snake_case** consistently (kebab-case is more readable in URLs).

## 3. Choose the Right HTTP Method

| Method   | Use Case                          | Idempotent? | Safe? |
|----------|-----------------------------------|-------------|-------|
| **GET**    | Retrieve resource(s)             | Yes         | Yes   |
| **POST**   | Create new resource              | No          | No    |
| **PUT**    | Replace entire resource          | Yes         | No    |
| **PATCH**  | Partial update                   | No*         | No    |
| **DELETE** | Delete resource                  | Yes         | No    |

\*PATCH can be made idempotent with proper implementation (e.g., using `If-Match` headers).

## 4. Idempotency – Critical for Reliability

Idempotency guarantees that repeating the same request produces the same result (no duplicate side effects).

**How to achieve it:**
- **PUT** and **DELETE** are naturally idempotent.
- **POST** is **not** idempotent by default → use one of these patterns:
  - **Idempotency-Key header** (recommended industry standard).
  - Use **unique client-generated IDs** in the request body.
  - Return appropriate success codes or `409 Conflict` for duplicates.
- Always document which endpoints are idempotent.

**Pro tip**: Idempotency is especially important for payment, order creation, and financial APIs.

## 5. HTTP Status Codes – Be Precise and Consistent

Use the correct status code **every time**. Developers rely on these for control flow.

- **Success**: `200 OK`, `201 Created`, `202 Accepted`, `204 No Content`
- **Client Errors**: `400 Bad Request`, `401 Unauthorized`, `403 Forbidden`, `404 Not Found`, `409 Conflict`, `422 Unprocessable Entity`, `429 Too Many Requests`
- **Server Errors**: `500 Internal Server Error`, `503 Service Unavailable`

**Best practice**: Return a **consistent error envelope** with `code`, `message`, and optional `details`.

## 6. Pagination – Don’t Return Everything

Prefer **cursor-based pagination** for scalability. Always include metadata (`total`, `limit`, `next_cursor`, `has_more`).

## 7. Security – Non-Negotiable

- Always use HTTPS.
- Prefer **OAuth 2.1** / OpenID Connect.
- Implement strict input validation, rate limiting, and proper authorization.
- Never expose secrets in URLs or responses.

## 8. Versioning Strategy

Prefer URL versioning (`/v1/users`, `/v2/users`). Maintain backward compatibility and deprecate gracefully.

## 9. Developing for the Latest OpenAPI Specification Standard

The **OpenAPI Specification (OAS)** is the industry-standard, machine-readable format for describing REST APIs. It serves as the **single source of truth** for your API contract, enabling automatic generation of documentation, client SDKs, server stubs, tests, and validation.

### Current Standard (as of 2026)
- **Latest version**: **OpenAPI 3.2.0** (released September 2025).
- It builds on OpenAPI 3.1 with improvements including:
  - First-class support for **streaming responses** (Server-Sent Events, JSON Lines, multipart streams) — essential for real-time and AI-powered APIs.
  - Better handling of **non-standard HTTP methods**, path templating, and example serialization.
  - Enhanced support for modern OAuth flows (including device authorization flow).
  - Full alignment with **JSON Schema Draft 2020-12** for richer, more expressive schemas.

**Recommendation**: Target **OpenAPI 3.2** for all new APIs. Most tooling now supports it fully, and it remains backward-compatible with 3.1.x.

### Key Best Practices for OpenAPI Development

- **Adopt a Design-First Approach** (strongly recommended):
  - Define the OpenAPI document **before** writing implementation code.
  - This prevents drift between spec and actual behavior.

- **Single Source of Truth**:
  - Store the `openapi.yaml` or `openapi.json` file in version control.
  - Generate documentation, SDKs, and even parts of your server code from it.
  - Name the root file `openapi.yaml` or `openapi.json`.

- **Structure and Organization**:
  - Use **components** extensively for reusable schemas, parameters, responses, and request bodies (`$ref` is your best friend).
  - Break large specs into multiple files and reference them (supported in 3.1+).
  - Organize operations with **tags** for logical grouping.
  - Always define **operationId** (unique, case-sensitive) — critical for code generation.
  - Provide rich **examples** for every schema, request, and response.

- **Leverage New 3.2 Features**:
  - Describe streaming endpoints properly (e.g., for real-time updates or LLM responses).
  - Use improved example objects and clearer semantics for complex content types.

- **Validation and Tooling**:
  - Validate your spec with tools like Spectral, Redocly, or the official OpenAPI validator.
  - Generate interactive docs with Swagger UI or Redoc.
  - Auto-generate client libraries (e.g., via OpenAPI Generator or language-specific tools).
  - Integrate into CI/CD to catch spec violations early.

- **Additional Tips**:
  - Keep titles, summaries, and descriptions concise but informative (under 50–100 characters where possible).
  - Specify the `servers` object (never leave it empty).
  - Include security schemes, external documentation links, and deprecation notices.
  - For large APIs, consider modular specs and tools that support multi-file references.

**Pro tip**: Treat your OpenAPI document like production code — lint it, review it in PRs, and evolve it with your API.

## 10. Additional Critical Best Practices

- **Filtering, Sorting, Searching**: Support via query parameters (e.g., `?status=active&sort=-created_at`).
- **Caching**: Use `ETag`, `If-None-Match`, and proper `Cache-Control` headers.
- **Consistency**: Same naming conventions, date formats (ISO 8601), and response envelopes across the entire API.
- **Performance**: Offer bulk operations, webhooks (instead of polling), and async endpoints (`202 Accepted`).
- **Backward Compatibility**: Add fields, never remove or change existing ones without versioning.

## 11. Common Pitfalls to Avoid

- Over-fetching/under-fetching (consider GraphQL as a complement for complex clients).
- Exposing internal implementation details.
- Returning `200 OK` with error bodies.
- Tight coupling between client and server URLs.
- Neglecting to keep the OpenAPI spec in sync with the implementation.

---

**Final Advice**:  
Treat your API like a **product** — and your OpenAPI specification as its **contract**. The best APIs (and their specs) feel invisible: developers love using them because they just work. By targeting the latest OpenAPI standard and following these practices, your APIs will be robust, secure, well-documented, and future-proof for years to come. 

Start with design-first + OpenAPI 3.2, and you'll thank yourself later.