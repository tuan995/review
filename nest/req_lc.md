# Nest Request Lifecycle & Core Concepts

## Building blocks
- Module groups related capabilities.
- Provider is managed by the dependency injection container.
- Controller receives requests and delegates business work to services/providers.
- Dependency injection reduces coupling and improves testability.

## Request lifecycle
A useful interview mental model is:

Request -> Middleware -> Guard -> Interceptor (before) -> Pipe -> Controller/Handler -> Service -> Interceptor (after) -> Response

Exception filters handle uncaught exceptions in their configured scope.

## Middleware
Useful for generic request-level work such as correlation IDs, logging or attaching context when authorization-specific decisions are not required.

## Guard
Answers whether a request may continue. Common examples are authentication, authorization and role/permission checks.

## Pipe
Transforms or validates incoming parameters/body before they reach the handler.

## Interceptor
Wraps handler execution. Useful for timing, logging, response transformation, caching and cross-cutting behavior around execution.

## Exception filter
Maps exceptions into controlled responses and can centralize error handling/logging.

## Dependency injection
Prefer constructor injection. Depend on abstractions where it helps isolate infrastructure. Avoid circular dependencies; usually they signal module/service boundaries that should be reconsidered.

## Scope
Providers are singleton by default. Request-scoped providers are useful only when request-specific state is truly necessary because they add lifecycle and allocation overhead.

## Interview questions
1. Module, controller and provider roles?
2. How does dependency injection work conceptually?
3. Middleware vs guard?
4. Guard vs interceptor?
5. Pipe vs middleware?
6. What is an exception filter?
7. Explain the request lifecycle.
8. Why avoid request scope unless necessary?
9. How do you handle circular dependencies?
10. How would you structure a large application by domain?

## Senior follow-ups
- Where would you implement authentication vs authorization?
- Where would you add correlation IDs and request timing?
- How would you make infrastructure adapters replaceable in tests?
- How do you prevent business logic from accumulating in controllers?
- How would you implement graceful shutdown for DB, Redis and queue consumers?
