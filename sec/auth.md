# Auth & API Protection

## Topics
- Authentication vs authorization
- JWT access token and refresh token
- OAuth 2.0 concepts
- HMAC request signing
- Webhook signature verification
- Token expiration and rotation
- Replay protection using timestamp and nonce
- Secret management
- Rate limiting
- Least privilege

## JWT
JWT is signed, not encrypted by default. Always validate signature, expiration, issuer/audience when applicable. Keep access tokens short-lived. Refresh tokens need stronger protection and rotation/revocation strategy.

## HMAC
Both parties share a secret. Sender signs a canonical representation of the request; receiver calculates the signature again and compares it securely. Include timestamp/nonce to reduce replay risk.

## Webhooks
Typical verification flow: preserve required raw payload, calculate expected signature using the shared secret, compare securely, reject invalid/stale requests, then process asynchronously when possible.

## Interview questions
1. Authentication vs authorization?
2. JWT vs session?
3. Why is JWT not automatically secure just because it is signed?
4. Access token vs refresh token?
5. How do you revoke JWTs?
6. What is HMAC and where would you use it?
7. How do you prevent replay attacks?
8. How do you verify a webhook?
9. Where should application secrets be stored?
10. How would you rotate a signing secret without downtime?

## Senior follow-ups
- What happens if a refresh token is stolen?
- How would you support key rotation while old requests are still in flight?
- Why should signature comparison avoid timing leaks?
- How do authentication and rate limiting interact?
