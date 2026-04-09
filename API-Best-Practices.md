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
  - **Idempotency-Key header** (recommended industry standard):
    ```http
    Idempotency-Key: 123e4567-e89b-12d3-a456-426655440000
    ```
    The server stores the key + result for a period (e.g., 24h) and returns the cached result for duplicates.
  - Use **unique client-generated IDs** in the request body (e.g., `order_id`).
  - Return `200 OK` or `201 Created` (with `Location` header) on success; `409 Conflict` if a duplicate is detected.
- Always document which endpoints are idempotent.

**Pro tip**: Idempotency is especially important for payment, order creation, and financial APIs.

## 5. HTTP Status Codes – Be Precise and Consistent

Use the correct status code **every time**. Developers rely on these for control flow.

### Success Codes
- `200 OK` – General success (GET, PUT, PATCH, DELETE)
- `201 Created` – Resource created (POST)
- `202 Accepted` – Request accepted but processing asynchronously
- `204 No Content` – Success with no body (e.g., DELETE)

### Client Errors (4xx)
- `400 Bad Request` – Invalid input (use with clear error message)
- `401 Unauthorized` – Missing or invalid credentials
- `403 Forbidden` – Authenticated but not authorized
- `404 Not Found` – Resource doesn’t exist
- `409 Conflict` – State conflict (e.g., duplicate idempotency key)
- `422 Unprocessable Entity` – Semantic validation errors
- `429 Too Many Requests` – Rate limiting

### Server Errors (5xx)
- `500 Internal Server Error` – Catch-all (never expose stack traces)
- `503 Service Unavailable` – Temporary overload / maintenance

**Best practice**: Return a **consistent error envelope**:

```json
{
  "error": {
    "code": "VALIDATION_FAILED",
    "message": "Request body is invalid",
    "details": [
      { "field": "email", "reason": "must be a valid email" }
    ]
  }
}
```

## 6. Pagination – Don’t Return Everything

Never return unbounded lists.

**Preferred approaches:**

1. **Cursor-based pagination** (recommended for scalability):
   ```http
   GET /users?cursor=eyJ2IjoxMjN9&limit=50
   ```
   Response includes `next_cursor` and `prev_cursor`.

2. **Offset/limit** (simpler but less efficient at scale):
   ```http
   GET /users?offset=100&limit=50
   ```

Always return **metadata**:
```json
{
  "data": [ ... ],
  "pagination": {
    "total": 1247,
    "limit": 50,
    "next_cursor": "abc123",
    "has_more": true
  }
}
```

## 7. Security – Non-Negotiable

- **Always use HTTPS** (TLS 1.2+).
- **Authentication**:
  - Prefer **OAuth 2.1** / **OpenID Connect** with JWTs.
  - Use short-lived access tokens + refresh tokens.
  - Support **API keys** only for machine-to-machine with IP restrictions.
- **Authorization**: Implement **RBAC** or **ABAC** at the resource level.
- **Input validation**:
  - Validate **every** field (size, format, allowed values).
  - Sanitize to prevent injection (use parameterized queries, ORMs).
- **Rate limiting** (per user/IP/key) – return `429` with `Retry-After` header.
- **CORS** – be explicit about allowed origins.
- **Security headers**: `Content-Security-Policy`, `Strict-Transport-Security`, etc.
- **Secrets** – never in URLs or logs. Use environment variables or secret managers.
- **Audit logging** for sensitive operations.

## 8. Versioning Strategy

**Preferred**: URL versioning (most explicit)
```
/v1/users
/v2/users
```

**Alternatives**:
- Header-based: `Accept: application/vnd.company.v2+json`
- Never break existing clients without notice (maintain old versions for 12–24 months).

## 9. Additional Critical Best Practices

- **Filtering, Sorting, Searching**:
  ```http
  GET /users?status=active&sort=-created_at&search=john
  ```
- **Caching**:
  - Use `ETag` + `If-None-Match` for conditional GETs.
  - Set proper `Cache-Control` headers.
- **Documentation**:
  - Use **OpenAPI 3.1** (Swagger) as the single source of truth.
  - Include examples for every endpoint and error case.
- **Consistency**:
  - Same date format (`ISO 8601`).
  - Same field naming (`snake_case` or `camelCase` — pick one).
  - Same response envelope for lists and singles.
- **Performance**:
  - Support **bulk operations** where it makes sense (`POST /users/bulk`).
  - Offer **webhooks** for event-driven updates instead of polling.
  - Use asynchronous endpoints (`202 Accepted` + callback URL) for long-running tasks.
- **Backward Compatibility**:
  - Add fields, never remove.
  - Deprecate gracefully with `Deprecation` header.

## 10. Common Pitfalls to Avoid

- Over-fetching / under-fetching → consider **GraphQL** as an alternative for complex UIs.
- Exposing internal IDs or implementation details.
- Using query strings for sensitive data.
- Returning 200 with error messages in the body (anti-pattern).
- Tight coupling between frontend and backend URLs.

---

**Final Advice**:  
Treat your API like a **product**. The best APIs feel invisible — developers love using them because they just work. Always ask: “Would I be happy consuming this API myself?”

Follow these practices and your APIs will be robust, secure, and a joy to work with for years to come.