# Access Control

Agent access control operates at three layers: the user's authorisation to invoke an agent, the agent's authorisation to invoke a tool, and the tool's authorisation to act on a downstream system. Collapsing these layers — treating the agent as a single identity with a single permission set — is the most common cause of over-privileged agents. An agent that inherits the union of permissions across all layers it touches has more authority than any individual user it serves, which violates least privilege and creates an outsized blast radius when the agent behaves unexpectedly.

## Three Layers

Each layer is a distinct authorisation boundary with its own enforcement mechanism. Passing a check at one layer does not grant access at the next.

```
  ┌──────────────┐     OIDC/OAuth      ┌──────────────┐
  │     User     │ ─────────────────── │  API Gateway  │
  └──────────────┘                     └──────┬───────┘
                                              │ JWT + RBAC check
                                       ┌──────▼───────┐
                                       │    Agent      │
                                       └──────┬───────┘
                                              │ ABAC policy (Cedar/OPA)
                                       ┌──────▼───────┐
                                       │  Tool Executor│
                                       └──────┬───────┘
                                              │ Scoped service credential
                                       ┌──────▼───────┐
                                       │ Downstream    │
                                       │ System        │
                                       └──────────────┘
```

### Layer 1: User to Agent

The user authenticates via the Experience plane using OIDC or OAuth. The resulting identity token is validated at the API gateway. The orchestration layer checks whether the user is authorised to invoke the requested agent. This is standard RBAC: the user has a role — for example, `claims_processor` — and the agent requires a role of `claims_processor` or above. Users may not invoke agents outside their role set, regardless of how the request is constructed. Rate limits are applied at this boundary on a per-user, per-agent basis to prevent runaway consumption.

### Layer 2: Agent to Tool

Each agent has a declared tool set, registered at deployment time and immutable at runtime. The Tool Executor verifies at invocation time that the calling agent is authorised to invoke the requested tool at the requested risk class. This is attribute-based access control: the policy decision depends on the agent's identity, the tool's risk class, the user's identity propagated via an on-behalf-of token, and the action being performed. A tool call that does not satisfy the policy is rejected before execution; the rejection is logged as a policy denial event in the audit trail. The model cannot instruct the executor to bypass this check — the policy runs outside the model's context.

### Layer 3: Tool to System

The downstream system — database, API, workflow engine — applies its own access controls to the credentials presented by the tool. The agent's service identity must be granted the minimum necessary permissions on the downstream system. Permissions are scoped to the specific resources the agent needs, not to the resource type. A service account granted read access to a named table rather than to the schema that contains it limits the damage if that identity is compromised or if the agent is manipulated into querying records it should not see.

## RBAC vs ABAC

The choice between role-based and attribute-based access control is not ideological — it is practical. RBAC is appropriate where the decision space is stable and auditable. ABAC is appropriate where the decision depends on context that varies per request. For agents, both are required at different layers.

| Concern | Recommended Approach | Rationale |
|---|---|---|
| User-to-agent authorisation | RBAC | User roles are stable; simple to audit |
| Agent-to-tool authorisation | ABAC | Contextual (risk class, user identity, action) |
| Tool-to-system authorisation | RBAC + least-privilege | Downstream systems often do not support ABAC natively |

In practice, ABAC at Layer 2 is implemented via Cedar or OPA policies evaluated by the Tool Executor. The policy receives the agent identity, the tool being invoked, the tool's risk class, the user's identity and role, and the action. It returns permit or deny. This evaluation happens synchronously before every tool call and adds negligible latency; the policy decision is logged regardless of outcome.

## Scoped Token Patterns

Never grant an agent a token with broader scope than it needs for the current request. For multi-step agents, generate scoped tokens per step rather than one broad token for the session. A step-scoped token is valid only for the tool invocations authorised within that step; it cannot be reused by a subsequent step or by a different agent that receives it via tool output. Step-scoped tokens limit the blast radius of a compromised step: an attacker who extracts the token from a step's tool call can invoke only the tools that step was authorised to call, for the duration of that step's execution window.

## Least-Privilege Checklist

The principle of least privilege applies at every layer. The table below states the specific application at each boundary. This checklist is verified at the Approval Gate security review and re-verified at annual recertification.

| Layer | Least-Privilege Application |
|---|---|
| User → Agent | User can only invoke agents their role permits |
| Agent → Tool | Agent can only invoke tools in its declared tool set |
| Agent → Model | Agent's IAM role has InvokeModel permission for specific models only |
| Agent → Secrets | Agent's IAM role has GetSecretValue for named secrets only |
| Tool → System | Service account has SELECT/INSERT on named tables only, not schema-wide |

Violations of least privilege discovered during the security review are blocking findings. An agent with an overly broad IAM role, access to secrets it does not use, or a tool set that includes capabilities not required by its documented purpose does not pass the Approval Gate. The correction must be made before the gate re-review; it is not acceptable to note the violation and proceed.
