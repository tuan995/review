# REST & GraphQL API Design

## REST
Design around resources, use HTTP methods and status codes consistently, validate inputs, paginate large collections, define a stable error shape, and make retryable write operations idempotent when possible.

## Pagination
Offset pagination is simple but can become expensive and inconsistent on frequently changing large datasets. Cursor/keyset pagination usually scales better when ordering is stable and indexed.

## GraphQL
Clients request the fields they need through a typed schema. Benefits include flexible reads and fewer endpoint-specific payloads. Trade-offs include resolver complexity, authorization, caching, query cost and the N+1 problem.

## N+1
A parent query returns N records and a resolver executes another query for each record. Typical fixes include batching/DataLoader, joins or preloading where appropriate.

## REST vs GraphQL
REST is often simpler operationally and works naturally with HTTP caching. GraphQL is useful for clients that need flexible related data but requires stronger controls for complexity, authorization and observability.

## Interview questions
1. PUT vs PATCH?
2. What makes an API idempotent?
3. Offset vs cursor pagination?
4. How do you version an API?
5. How do you design API error responses?
6. REST vs GraphQL trade-offs?
7. What is GraphQL N+1?
8. How does DataLoader help?
9. How would you limit expensive GraphQL queries?
10. How do you prevent duplicate processing when a client retries a POST?

## Senior follow-ups
- How would you evolve an API without breaking old clients?
- Where should authorization happen in GraphQL?
- How would you trace one request across multiple downstream services?
- How would you design bulk APIs for partial failure?
