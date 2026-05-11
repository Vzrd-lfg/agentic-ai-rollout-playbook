# Infrastructure Patterns

This directory documents the infrastructure patterns that support the five-plane architecture described in this playbook. No IaC modules are published here — the nine module shapes described below are documented as interfaces, not implementations. The reasoning for this boundary is explained in the final section. See DISCLAIMER.md for the broader publication policy.

---

## Account and Project Topology

The control plane (audit, identity, cost tracking, secrets management) runs in a dedicated account or project, isolated from the application zones. Application zones are separated by environment (non-production, production) and, where data residency requires it, by region. The blast radius of a compromised application zone does not extend to the control plane: application-zone roles may write audit events to control-plane resources but cannot modify or delete them, and the audit bucket is configured with Object Lock (AWS) or Bucket Lock (GCP) in compliance mode so that even the platform team cannot remove records before the retention period expires.

```
  ┌──────────────────────────────────────────────────────┐
  │  Control Account / Project                            │
  │  Audit Store │ Secrets Manager │ IAM │ Cost Tracking │
  └──────────────────┬───────────────────────────────────┘
                     │  cross-account role / Workload Identity
           ┌─────────┴─────────┐
           │                   │
  ┌────────▼────────┐  ┌───────▼──────────┐
  │  Non-Production │  │   Production     │
  │  Account/Project│  │  Account/Project  │
  │  (dev, staging) │  │  (live traffic)   │
  └─────────────────┘  └──────────────────┘
```

---

## Network Boundaries

There is no public-internet egress from the application zone. Model API calls (Bedrock, Vertex AI) route via AWS PrivateLink or GCP Private Service Connect. Tool services (databases, internal APIs) run behind internal endpoints reachable only from within the enterprise network. All traffic between the orchestration plane and the model plane, and between the orchestration plane and the data and tool plane, stays within the enterprise network boundary. Flow logs are retained and audited to verify that no agent subnet establishes connections to public model endpoint IP ranges.

```
  Enterprise Network
  ┌──────────────────────────────────────────────────────────┐
  │                                                          │
  │  ┌────────────────────────────────────────────────────┐  │
  │  │  Application VPC / VPC-SC Perimeter                 │  │
  │  │                                                    │  │
  │  │  Orchestration Plane ──► PrivateLink/PSC ──►        │  │
  │  │  (ECS / Cloud Run)        Bedrock / Vertex          │  │
  │  │                                                    │  │
  │  │  Tool Executor ──► Internal Endpoint ──►           │  │
  │  │                    Database / API / Workflow        │  │
  │  │                                                    │  │
  │  │  ◄── No public internet egress ──►                 │  │
  │  └────────────────────────────────────────────────────┘  │
  │                                                          │
  │  Control Plane VPC ──► S3/GCS (audit, separate bucket)  │
  └──────────────────────────────────────────────────────────┘
```

---

## Per-Plane Infrastructure Responsibilities

| Plane | Infrastructure Components | Key Controls |
|---|---|---|
| Experience Plane | API gateway (APIGW / Cloud Endpoints), WAF, CDN | TLS termination, DDoS protection, rate limiting |
| Orchestration Plane | Container runtime (ECS Fargate / Cloud Run), load balancer | Auto-scaling, health checks, circuit breaker |
| Model Plane | PrivateLink / PSC endpoint to Bedrock / Vertex | No public egress; endpoint policy restricts callers |
| Data & Tool Plane | VPC endpoints for databases; internal service discovery | Network ACLs; service-to-service mTLS |
| Control Plane | S3 / GCS (audit, Object Lock); Secrets Manager / Secret Manager; IAM | Append-only audit bucket; secrets rotation; least-privilege roles |

---

## Nine IaC Module Shapes

The nine modules below describe the interface — what resources they create, what inputs they require, and what outputs they expose — rather than the implementation. They are documented here as the target architecture; the IaC implementation is not published. A competent platform team can implement each module from this interface description combined with the patterns in the governance and integrations sections of this playbook.

| Module | Creates | Key Inputs | Key Outputs |
|---|---|---|---|
| `vpc` | VPC, subnets, PrivateLink/PSC endpoints, security groups | Region, CIDR ranges, endpoint services | VPC ID, subnet IDs, endpoint IDs |
| `iam` | IAM roles for each agent, cross-account trust policies | Agent IDs, tool permission matrix | Role ARNs / service account emails |
| `audit` | Audit S3/GCS bucket with Object Lock, KMS key, log group | Retention tiers, encryption key | Bucket name, log group ARN |
| `runtime` | Container service (ECS / Cloud Run), task definitions | Container images, scaling policy | Service ARNs, endpoints |
| `tools` | Tool Registry (DynamoDB / Firestore), tool endpoint configuration | Tool definitions with risk classes | Registry table ARN |
| `secrets` | Secrets Manager entries, rotation schedules | Secret names, rotation Lambda | Secret ARNs |
| `eval` | Eval runner Lambda/Cloud Function, S3/GCS bucket for results | Golden set location, agent endpoints | Runner ARN, results bucket |
| `observability` | CloudWatch/Cloud Monitoring dashboards, OTel collector, alert rules | Agent IDs, alert thresholds | Dashboard URLs, alert policy IDs |
| `policy` | Cedar policy store / OPA deployment, policy bundle S3/GCS | Policy documents | Policy store ARN |

---

## What Is Not Here

The IaC module implementations, the Cedar policy packs, the Grafana dashboard JSON, and the alert rule sets are not published in this repository. The reasoning is explained in DISCLAIMER.md: published IaC encourages copy-paste deployment before the security review, threat modelling, and change-management work that production demands has been completed; the most consequential configuration details (IAM trust policies, KMS key boundaries, audit-bucket object lock settings) are organisation-specific and cannot be sensibly templated; and the patterns documented here are the durable artefact, not any particular implementation of them.
