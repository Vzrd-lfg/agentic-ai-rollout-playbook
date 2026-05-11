# Identity and Secrets Management

Every agent invocation involves three distinct identities: the human or system account that initiated the request, the service identity of the agent process itself, and the on-behalf-of token that the agent presents when calling tools requiring user-level access. Conflating these three — the most common failure mode in early agentic deployments — produces over-privileged agents whose actions cannot be attributed to specific users, audit trails that cannot answer regulatory questions, and access control that cannot be revoked selectively.

## Three Identities

### User Identity

The human or system account that initiated the request. It enters the system as an OIDC token issued by the enterprise identity provider, is validated at the API gateway, and is propagated as a signed identity assertion through the orchestration spine. It is never stored; it travels as a claim in the request context and is re-verified on each tool call by the Tool Executor's policy engine. User identity is the basis for evaluating user-scoped permissions: whether this user is permitted to invoke this agent, and whether this user has access to the data the tool will read or write.

### Agent Service Identity

The IAM role or service account assigned to the agent process itself. The agent uses this identity to authenticate to the model API, to the vector store, and to internal platform services. It carries the minimum permissions required for the agent's declared tool set — not a general-purpose role that happens to cover the tools. Each agent receives its own service identity; roles are not shared across agents. Sharing a role makes it impossible to revoke one agent's access without affecting others, impossible to audit which agent called which downstream service, and impossible to demonstrate to a regulator that one agent's permissions are scoped to its function.

### On-Behalf-Of Token

When the agent calls a tool that requires user-level access — reading a document the user is permitted to see, writing to a system the user is authorised to modify — it presents an on-behalf-of token that combines the agent service identity with the user identity. The downstream system receives both: it knows that the agent is the technical caller, and it knows on whose behalf the call is made. This preserves the source system's access control model without granting the agent full user permissions. The on-behalf-of token is scoped to the current request, not cached across requests, and is audited as a compound identity event.

## Secrets Management

Secrets — API keys, database passwords, service credentials, signing keys — are never hardcoded in agent source code and never passed as environment variables in production deployments. They are stored in a managed secrets service (AWS Secrets Manager or GCP Secret Manager), retrieved at agent startup or per-request depending on the credential type, and rotated on a schedule that does not require agent restarts. The agent's service identity is granted `GetSecretValue` permission for the specific secret ARNs or resource paths it needs and nothing else; it cannot enumerate secrets or access credentials belonging to other agents. Rotation events are consumed transparently: the agent fetches the current secret value on each use, so a rotated credential becomes effective at the next fetch without a deployment.

## Short-Lived Credentials

Where possible, we prefer short-lived credentials over long-lived API keys. IAM `AssumeRole` with a session TTL of 15–60 minutes, or Workload Identity Federation on GCP, produces credentials that expire automatically and require no explicit revocation if an agent process is compromised. A compromised short-lived credential is only useful until it expires; a compromised long-lived key requires immediate detection and manual rotation to limit the blast radius. Long-lived keys are acceptable only for external services that do not support federated authentication, and they must be registered in the secrets manager with a mandatory rotation schedule.

## Secrets Rotation Pattern

```
  Secrets Manager          Agent Process          Downstream Service
        │                       │                        │
        │◄── GetSecretValue ─────│                        │
        │─── secret_value ──────►│                        │
        │                       │── authenticate ────────►│
        │                       │◄── session token ───────│
        │                       │                        │
  [rotation event]              │                        │
        │                       │                        │
        │◄── GetSecretValue ─────│  (next request)        │
        │─── new_secret_value ──►│                        │
        │                       │── re-authenticate ─────►│
```

The agent process does not hold a credential object with a fixed expiry that it manages internally. It fetches the current value from the secrets service on each use. The rotation event in the secrets service propagates automatically: the next `GetSecretValue` call returns the new value, the agent re-authenticates to the downstream service, and the old credential is invalidated by the downstream service's session management. No deployment, no restart, no coordination window.
