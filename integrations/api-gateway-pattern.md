# API Gateway Pattern

All traffic to the orchestration spine enters through a single API gateway. The gateway normalises requests, enforces authentication, and injects observability context before any agent logic executes. This centralisation is not an architectural nicety — it is the control point that makes central auditing, cost tracking, and model swaps operationally tractable. Bypassing it, even for internal callers, breaks the audit trail and distributes guardrail logic across surfaces that will each implement it differently.

## Three Ingress Lanes

Every inbound request belongs to exactly one of three lanes. The lane determines the authentication mechanism, the token validation path, and how the user identity is resolved and propagated. A request that cannot be classified into a lane is rejected before any agent logic executes.

| Lane | Trigger | Authentication | Examples |
|---|---|---|---|
| User Lane | Human-initiated | OIDC/OAuth 2.0 (PKCE flow) | Web UI, Slack app, mobile |
| System Lane | Machine-to-machine | mTLS or OAuth 2.0 client credentials | Upstream microservices, scheduled jobs |
| Webhook Lane | External event | HMAC signature verification | GitHub, Jira, Stripe events |

## Gateway Flow

```
  Inbound Request
        │
        ▼
  ┌─────────────┐
  │  Auth Check  │── fail ──► 401/403 response
  │  (per lane)  │
  └──────┬───────┘
         │ pass
         ▼
  ┌─────────────┐
  │ Rate Limiter │── exceeded ──► 429 response
  └──────┬───────┘
         │ within limit
         ▼
  ┌─────────────┐
  │  Request     │  Normalise to internal schema
  │ Normaliser  │  Attach: user identity, request ID, timestamp
  └──────┬───────┘
         │
         ▼
  ┌─────────────┐
  │  Trace       │  Inject W3C TraceContext headers
  │  Injector    │  Start root span
  └──────┬───────┘
         │
         ▼
  Orchestration Spine
```

## Authentication Per Lane

The User Lane presents an OIDC ID token issued by the enterprise identity provider following the PKCE flow. The gateway validates the token signature against the IdP's JWKS endpoint, checks expiry, and extracts the `sub`, `email`, `groups`, and any custom role claims. These claims are forwarded to the orchestration spine as a signed internal identity assertion; the raw token is not passed downstream. The Tool Executor's policy engine uses the propagated identity to evaluate user-scoped permissions on each tool call.

The System Lane authenticates machine callers via mutual TLS or the OAuth 2.0 client credentials grant. For mTLS, the gateway validates the client certificate against the internal CA and extracts the service identity from the certificate's Common Name or SAN. For client credentials, it validates the client secret and issues a short-lived access token. The resolved service identity — not the certificate or credential — is carried into the orchestration layer, where it determines which agents and tools the calling service is permitted to invoke.

The Webhook Lane accepts inbound events from external systems. Authentication is HMAC signature verification: the gateway recomputes the expected signature over the raw request body using the shared secret registered for that webhook source, and rejects any request whose signature does not match within a constant-time comparison. Replays are suppressed by checking the event timestamp against a configurable tolerance window and by deduplicating event IDs against a short-lived cache. Verified events are placed on an internal queue; a worker process attaches a synthetic user identity derived from the webhook source registration before forwarding to the orchestration spine.

## Rate Limiting

Rate limiting is applied per user identity — not per IP address — per agent, and per risk class. Calls that invoke IRREVERSIBLE-class tools carry a lower per-minute ceiling than READ-class calls, because the cost of a runaway loop that modifies external state is categorically different from one that only reads. Limits are configurable per agent and per organisation unit without a deployment. The gateway communicates current limit state to callers via standard response headers: `X-RateLimit-Limit` carries the ceiling, `X-RateLimit-Remaining` carries the remaining budget for the current window, and `Retry-After` is set on 429 responses to indicate when the window resets.

## Request Normalisation

Before a request reaches the orchestration spine, the gateway produces a normalised internal request envelope that all downstream components operate on exclusively. The envelope contains: `request_id` (a UUID generated at the gateway, used for idempotency and correlation), `user_identity` (the resolved identity assertion from the auth check), `agent_id` (the target agent, parsed from the request path or body), `input_payload` (the caller-supplied content, schema-validated against the agent's declared input contract), `timestamp` (the gateway's wall-clock time at receipt, in ISO 8601), `source_lane` (one of `user`, `system`, or `webhook`), and `trace_id` (the W3C TraceContext `traceparent` value injected by the trace injector). No downstream component receives raw HTTP requests; they receive only this envelope, which means authentication, schema validation, and trace context are guaranteed to be present by the time any agent logic runs.
