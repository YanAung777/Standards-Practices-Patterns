# Enterprise AI Guardrails --- Detailed Design and Implementation Setup

**Document type:** Solution Architecture and Technical Design\
**Status:** Reference design\
**Version:** 1.1\
**Date:** 23 October 2026\
**Audience:** Enterprise architects, AI engineers, application
developers, security engineers, platform engineers, risk and assurance
teams

------------------------------------------------------------------------

## 1. Executive Summary

An AI guardrail is an enforcement layer that evaluates an AI interaction
against defined security, safety, data-protection, quality, and
application requirements. It determines whether an input may be
processed, which context and tools the model may use, whether the
generated output is acceptable, and what action to take when a check
fails.

The LLM generates candidate content; it must not be the sole authority
deciding whether its own response is safe or permitted. The application
or guard service enforces policy independently.

This design adopts a layered pattern:

1.  Validate and classify the incoming request.
2.  Authenticate the caller and authorise requested capabilities.
3.  Apply input safety and prompt-injection controls.
4.  Retrieve only authorised context, if retrieval is required.
5.  Invoke an approved model through a controlled interface.
6.  Validate the model output against a versioned contract.
7.  Apply security, privacy, grounding, and business-policy checks.
8.  Execute a defined pass, retry, transform, refuse, or human-review
    action.
9.  Return only approved content and emit a redacted audit event.

The reference architecture is suitable for chat, summarisation, document
extraction, retrieval-augmented generation (RAG), and AI-assisted
workflows. Applications that permit transactions, external
communications, code execution, or other consequential actions require
additional authorisation and human approval controls.

## 2. Objectives and Non-Goals

### 2.1 Objectives

-   Enforce structured output contracts and application-specific
    business rules.
-   Reduce harmful, disallowed, irrelevant, malformed, and
    sensitive-data-leaking responses.
-   Protect against prompt injection and untrusted retrieved content.
-   Ensure tools and data are accessible only under explicit
    authorisation.
-   Support auditable, versioned policies and repeatable testing.
-   Limit retries, latency, cost, and failure amplification.
-   Provide safe, predictable behaviour when a validator, model, or
    dependency fails.

### 2.2 Non-goals

-   Guarantee that every model statement is true.
-   Replace identity management, API security, application
    authorisation, data governance, or human accountability.
-   Treat an LLM-based judge as a deterministic security boundary.
-   Use output filtering as a substitute for restricting model tool
    permissions.
-   Automatically repair every failure. Some failures require refusal or
    human review.

## 3. Design Principles

1.  **Default deny for privileged actions.** A model response must not
    grant itself access or authorise an operation.
2.  **Validate at trust boundaries.** Validate inputs, retrieved
    content, model outputs, tool arguments, and tool results.
3.  **Deterministic controls first.** Prefer schema validation,
    allowlists, type checks, and explicit business rules over
    model-based judgement when feasible.
4.  **Layered controls.** No single classifier, prompt, or validator is
    sufficient.
5.  **Least privilege.** Tools and retrieval services receive only the
    permissions required for the current operation.
6.  **Bounded recovery.** Retry only selected failure classes, with
    strict attempt and time limits.
7.  **Safe degradation.** If a mandatory security check is unavailable,
    do not silently bypass it.
8.  **Data minimisation.** Send the model only the information needed
    for the task.
9.  **Versioned policy.** Every decision must be attributable to a
    policy and schema version.
10. **Separation of duties.** Model generation, policy enforcement,
    privileged tool execution, and audit administration should have
    appropriately separated responsibilities.

## 4. Reference Architecture

### 4.1 Logical components

  --------------------------------------------------------------------------
  Component                  Responsibility          Trust boundary /
                                                     requirement
  -------------------------- ----------------------- -----------------------
  Client / application       Collects the request    Never holds
                             and presents approved   model-provider secrets
                             output                  in browser code

  API gateway                Authentication          Reject unauthenticated
                             integration, rate       or oversized requests
                             limits, request-size    
                             limits, routing         

  Guard service /            Coordinates policy,     Core enforcement point;
  orchestration layer        validators, model       authenticated
                             calls, retries, and     service-to-service
                             response decisions      calls

  Input validator            Checks type, size,      Reject malformed input
                             encoding, required      before model invocation
                             fields, and permitted   
                             request modes           

  Identity and policy        Applies user, tenant,   Must use trusted
  evaluator                  role, purpose, and      identity claims, not
                             capability restrictions user-supplied text

  Input safety and injection Detects disallowed      A classifier is
  checks                     requests and suspicious supplementary; never
                             instructions            rely on it alone

  Retrieval service          Retrieves authorised    Enforce document-level
  (optional)                 documents and evidence  access control before
                                                     context reaches the
                                                     model

  Prompt/template manager    Supplies approved,      Prevent arbitrary
                             versioned instructions  untrusted content from
                                                     overriding system
                                                     policy

  Model adapter              Invokes approved model  Secrets held
                             endpoints and           server-side; enforce
                             normalises provider     timeout and quota
                             responses               

  Output schema validator    Checks JSON/schema,     Deterministic
                             types, required fields, validation
                             bounds, and             
                             enumerations            

  Semantic/business          Checks application      Rules should be
  validator                  invariants and domain   explicit, testable, and
                             rules                   versioned

  Safety/privacy/grounding   Check policy            Independent checks;
  validators                 violations, sensitive   record result and
                             data, evidence support, confidence where
                             and citations as        available
                             appropriate             

  Action policy engine       Selects pass, retry,    No implicit pass when
                             transform, refuse, or   required checks fail
                             human review            

  Tool broker (optional)     Executes approved tools Validate tool name and
                             after separate          arguments; use narrow
                             authorisation           credentials

  Audit/telemetry pipeline   Records decision        Redaction, access
                             metadata, failures,     controls, retention and
                             latency, and metrics    integrity controls

  Policy administration      Manages schemas, policy Restricted access and
                             versions, thresholds,   change audit
                             and approvals           
  --------------------------------------------------------------------------

### 4.2 Mermaid architecture diagram

``` mermaid
flowchart TD
    A[Client / Application] --> B[API Gateway: Auth, Rate Limits]
    B --> C[Request Validation]
    C --> D[Identity and Capability Authorisation]
    D --> E[Input Safety and Injection Checks]
    E --> F{Input permitted?}
    F -- No --> R[Safe Refusal / Rejection]
    F -- Yes --> G[Authorised Retrieval, if required]
    G --> H[Approved Prompt Assembly]
    H --> I[Model Adapter]
    I --> J[LLM Candidate Output]
    J --> K[Schema and Type Validation]
    K --> L[Business and Semantic Rules]
    L --> M[Safety, Privacy and Grounding Checks]
    M --> N{Policy decision}
    N -- Pass --> O[Return Approved Output]
    N -- Retryable failure --> P[Bounded Retry / Repair]
    P --> I
    N -- Refuse --> R
    N -- Human review --> S[Review Queue]
    N -- Dependency failure --> T[Safe Fallback / Fail Closed]
    O --> U[Redacted Audit Event]
    R --> U
    S --> U
    T --> U
```

The diagram is logical, not a deployment topology. Production
deployments should add network segmentation, private connectivity where
required, secret management, workload identity, service health
monitoring, and environment separation.

## 5. End-to-End Request Lifecycle

### Stage 1 --- Receive and authenticate

-   Accept requests only through the approved application/API entry
    point.
-   Validate token issuer, audience, signature, expiry, tenant, and
    required scopes/roles.
-   Derive the caller identity from trusted authentication context.
-   Apply request-size, concurrency, per-user, per-tenant, and global
    rate limits.
-   Generate or propagate a correlation ID.
-   Reject invalid identity or malformed requests before calling the
    model.

### Stage 2 --- Validate and classify input

-   Validate content type, character encoding, required fields, maximum
    length, and permitted modes.
-   Detect unexpected binary content, malformed structured input, and
    unsupported attachments.
-   Apply data classification and identify whether sensitive information
    is necessary for the task.
-   Use a policy-defined input safety check for disallowed requests.
-   Treat retrieved documents, user-uploaded files, webpages, and tool
    output as untrusted data.
-   Identify prompt-injection attempts where relevant. Detection should
    inform risk handling but is not a complete defence.

### Stage 3 --- Authorise context and tools

-   Authorise each retrieval query using the caller's actual
    permissions.
-   Filter documents before they enter the model context; do not rely on
    the model to ignore unauthorised documents.
-   Keep system policy separate from user instructions and
    quoted/retrieved content.
-   Explicitly label untrusted content in prompts, while recognising
    that delimiters alone do not prevent injection.
-   Do not expose tools unless the current task and caller are
    authorised to use them.
-   For consequential operations, require separate server-side
    authorisation and, where appropriate, human confirmation.

### Stage 4 --- Build the model request

The request should include only what is needed:

-   Approved system/developer instructions.
-   User request after input validation.
-   Authorised, relevant context.
-   Output schema or format requirements.
-   Correlation and policy metadata held outside user-controlled prompt
    text where possible.
-   Model parameters appropriate to the task.

Do not place secrets, access tokens, internal credentials, or
unnecessary personal information in prompts. Use a versioned prompt
template and record its identifier.

### Stage 5 --- Invoke the model

-   Use an approved model endpoint and server-side credentials or
    workload identity.
-   Apply a request timeout, token budget, concurrency limit, and retry
    policy.
-   Handle provider errors, malformed responses, empty output, refusal
    markers, and content-filter outcomes explicitly.
-   Do not retry non-retryable errors indiscriminately.
-   Ensure retries cannot multiply an external side effect. Model
    generation retries should not automatically replay tool actions.

### Stage 6 --- Validate candidate output

Run validators in a defined order, normally from low-cost deterministic
checks to more expensive checks:

1.  Response exists and is within size limits.
2.  Content type and syntax are valid.
3.  Schema, types, required fields, enums, and numeric/string bounds are
    valid.
4.  Cross-field and business invariants hold.
5.  Safety and disallowed-content checks pass.
6.  Sensitive-data policy is satisfied.
7.  Claims are supported by authorised evidence when the use case
    requires grounding.
8.  Citations, references, and source IDs are well-formed and refer to
    permitted sources.
9.  Tool calls, if any, are separately authorised and their arguments
    validated.

A schema-valid response may still be unsafe, false, biased, ungrounded,
or unauthorised. Validation layers must therefore be selected according
to the use case's risk.

### Stage 7 --- Decide and recover

The policy engine combines validator results and severity into an
explicit decision. Do not allow an average score to cancel a critical
security failure.

-   **Pass:** all mandatory checks passed.
-   **Retry:** a retryable failure may be corrected by generating a new
    candidate.
-   **Transform:** a deterministic transformation can safely remove or
    normalise a known issue without changing meaning in a risky way.
-   **Refuse:** the request or response violates policy, or safe
    processing is not possible.
-   **Human review:** uncertainty or impact exceeds the application's
    permitted automated decision boundary.
-   **Fail closed:** a mandatory security validator, authorisation
    service, or policy dependency is unavailable.

A repair prompt must not instruct the model to bypass policy. Repaired
output must pass the complete required validation pipeline again.

### Stage 8 --- Return output

-   Return only the final approved response.
-   Use a stable, documented API response contract.
-   Avoid returning internal validator details that would help attackers
    tune bypass attempts.
-   If blocked, return a concise, safe message and a suitable error
    category.
-   For asynchronous human review, return a review/pending status rather
    than implying the action succeeded.

### Stage 9 --- Audit and monitor

Record the decision and necessary metadata, including: - Correlation ID
and timestamp. - Application, tenant, and pseudonymous caller identifier
where appropriate. - Model deployment/version and prompt-template
version. - Schema and policy versions. - Validator outcomes and failure
categories. - Action selected, retry count, latency, token usage, and
dependency status. - Tool names and authorised action IDs, where
applicable.

Do not log raw prompts or responses by default if they may contain
personal, confidential, or secret data. If content logging is justified,
document the purpose, access, redaction, retention, and approval.

## 6. Output Contract Design

### 6.1 Contract requirements

Each task should have a versioned output contract. Define: - Required
and optional fields. - Primitive and nested types. - Enumerated
values. - Length and numeric bounds. - Date/time formats and timezone
rules. - Unknown-field behaviour. - Cross-field constraints. -
Null/empty-value semantics. - Maximum output size. - Version
compatibility and migration rules.

Use JSON Schema or a language-native validator such as Pydantic. Treat
model-generated values as untrusted even after parsing.

### 6.2 Example Pydantic contract

``` python
from typing import Literal
from pydantic import BaseModel, ConfigDict, Field

class GuardedAnswer(BaseModel):
    model_config = ConfigDict(extra="forbid", str_strip_whitespace=True)

    status: Literal["answered", "insufficient_evidence", "refused"]
    answer: str = Field(min_length=1, max_length=8000)
    confidence: float = Field(ge=0.0, le=1.0)
    citations: list[str] = Field(default_factory=list, max_length=20)
```

This is an illustrative contract, not a complete security policy. The
application must validate that each citation identifies an authorised
source and that the response's claims are supported when grounding is
required. A model-generated confidence score is not a calibrated
probability unless independently evaluated.

### 6.3 Validation sequence

``` python
def validate_candidate(raw_output, context, policy):
    parsed = parse_response(raw_output)
    structured = validate_schema(parsed, policy.output_schema)
    validate_business_rules(structured, context, policy)
    validate_safety(structured, policy)
    validate_sensitive_data(structured, policy)
    validate_grounding(structured, context, policy)
    return structured
```

Each function above represents an explicit implementation
responsibility. Exceptions should be mapped to controlled failure
categories; do not convert unexpected exceptions into success.

## 7. Failure Actions and Decision Matrix

  ---------------------------------------------------------------------------
  Failure          Default action                     Retry? Notes
  ---------------- ------------------- --------------------- ----------------
  Invalid JSON or  Retry/repair once               Sometimes Revalidate
  schema           or twice within                           complete output
                   configured limit                          after repair

  Missing required Retry with                        Limited If repeated,
  field            structured error                          refuse or return
                   feedback                                  safe fallback

  Business         Reject or                    Only if safe Never weaken the
  invariant        deterministic                             invariant to
  violation        repair                                    make output pass

  Disallowed       Refuse or apply                Usually no Do not ask the
  content          approved safety                           model to restate
                   handling                                  prohibited
                                                             content

  Sensitive-data   Block, redact where               Limited Consider
  disclosure       safe, record event                        incident
                                                             handling for
                                                             actual exposure

  Unsupported      Ask for                           Limited Do not fabricate
  factual claim    evidence-grounded                         citations
                   revision or state                         
                   uncertainty                               

  Prompt injection Ignore untrusted        Not automatically Ensure tools and
  detected         instructions;                             data remain
                   constrain or refuse                       independently
                   as policy dictates                        restricted

  Authorisation    Deny                                   No Never retry by
  failure                                                    broadening
                                                             permissions

  Mandatory        Fail closed for                        No Safe static
  validator        protected                                 fallback may be
  unavailable      operations                                appropriate

  Model timeout or Bounded retry or                  Limited Respect
  rate limit       temporary service                         deadlines and
                   error                                     provider retry
                                                             guidance

  High-impact      Human review        No automated approval Use explicit
  ambiguous result                                           review queue and
                                                             approval record
  ---------------------------------------------------------------------------

### 7.1 Retry controls

-   Set a low, explicit maximum attempt count, for example two total
    model attempts.
-   Use an overall request deadline; each retry consumes the same
    budget.
-   Retry only failures that a new generation could reasonably correct.
-   Do not retry authentication, authorisation, policy-denial, or
    critical safety failures.
-   Apply exponential backoff with jitter for transient infrastructure
    errors.
-   Cap tokens and cost per request.
-   Avoid including full sensitive validator output in repair prompts.
-   Track retries and alert on unusual increases.

The example count is a starting configuration, not a universal
requirement. Set limits based on latency, cost, risk, and workload.

### 7.2 Meaning of common actions

-   **Re-ask:** Generate a new candidate using constrained feedback.
-   **Filter:** Remove or block content only where the transformation is
    reliable and does not create a misleading answer.
-   **Fix:** Apply deterministic normalisation or a bounded
    structured-output repair.
-   **Refrain:** Refuse or state that the system cannot safely answer.
-   **No-op:** Continue only if the relevant check is explicitly
    optional for the task. A no-op must never override a mandatory
    control.

## 8. Security Design

### 8.1 Prompt injection

-   Treat user input, retrieved documents, web pages, email, code, and
    tool output as untrusted.
-   Separate instructions from data and clearly delimit untrusted
    content.
-   Do not let retrieved content alter system policy, tool permissions,
    or access controls.
-   Use retrieval-time access controls and narrow tool capabilities.
-   Require explicit server-side checks before any external side effect.
-   Test direct and indirect injection, including instructions embedded
    in documents and metadata.
-   Do not assume a prompt-injection classifier or prompt wording can
    eliminate this risk.

### 8.2 Tool and function calling

For every tool call: 1. Check that the tool is enabled for this
application and task. 2. Check that the authenticated user is authorised
for the requested operation. 3. Validate arguments against a strict
schema and allowlists. 4. Apply resource-level authorisation on the
target object. 5. Use a dedicated least-privilege service identity. 6.
Enforce timeouts, rate limits, idempotency, and transaction boundaries.
7. Require confirmation or human approval for consequential actions. 8.
Validate tool results before returning them to the model or user. 9.
Audit both the request and the outcome without logging secrets.

Never treat a model-generated statement such as "the user approved this"
as proof of approval.

### 8.3 Sensitive information and privacy

-   Classify inputs and outputs according to organisational data policy.
-   Minimise personal and confidential data in prompts and logs.
-   Detect secrets, credentials, regulated identifiers, and restricted
    data where relevant.
-   Redact or block output according to the applicable policy.
-   Restrict access to logs and review queues.
-   Define retention and deletion rules for prompts, outputs,
    embeddings, traces, and caches.
-   Review provider data-handling, residency, retention, and training
    settings before production use.
-   Ensure redaction does not silently change critical meaning; prefer
    block-and-review when safe transformation is uncertain.

### 8.4 Grounding and hallucination

-   Use retrieval or authoritative APIs for answers that require current
    or controlled facts.
-   Enforce access control before retrieval results enter the model
    context.
-   Require citations or source identifiers for claims where
    appropriate.
-   Verify that cited sources exist, are authorised, and support the
    claims.
-   Distinguish sourced facts, inference, and uncertainty.
-   Use abstention when evidence is insufficient.
-   Evaluate grounding with representative test data; an LLM judge alone
    is not proof.

### 8.5 Bias, safety, and content policy

-   Define use-case-specific unacceptable outcomes.
-   Evaluate false positives and false negatives across representative
    user groups and language patterns.
-   Provide appeal or human-review paths where appropriate.
-   Use more than one signal for high-risk decisions.
-   Do not use model output alone to make consequential eligibility,
    employment, financial, medical, or legal decisions without the
    required governance and qualified human oversight.

### 8.6 Secrets and infrastructure

-   Store provider keys in a secret manager or use supported workload
    identity.
-   Do not put keys in frontend code, prompts, source control, or
    telemetry.
-   Use TLS for service communication and encryption at rest.
-   Apply network restrictions and private connectivity where required
    by the deployment.
-   Separate development, test, and production identities, policies,
    data, and endpoints.
-   Apply least privilege to policy editing, deployment, log access, and
    incident response.

## 9. API and Service Contract

An illustrative response contract:

``` json
{
  "request_id": "opaque-correlation-id",
  "status": "approved",
  "result": {
    "status": "answered",
    "answer": "Validated response text",
    "citations": []
  },
  "policy_version": "guard-policy-1.0"
}
```

Possible top-level statuses: - `approved` - `refused` -
`review_required` - `temporarily_unavailable` - `invalid_request`

Do not return internal chain-of-thought or private model reasoning.
Expose only the result, concise explanation, citations where applicable,
and safe error details.

HTTP status codes should be consistent with the wider API standard. For
example, malformed requests may use 400, unauthenticated requests 401,
unauthorised requests 403, rate limits 429, and transient dependency
failures 503. A policy refusal can be represented using the
organisation's documented application-level error contract.

## 10. Policy Configuration

Keep policy external to application code where practical, but protect it
with access controls and change management. Example conceptual
configuration:

``` yaml
policy_id: customer-support-answer
version: "1.0"
output:
  schema: customer-support-answer-v1
  max_characters: 8000
checks:
  schema_validation: required
  business_rules: required
  safety_validation: required
  sensitive_data_validation: required
  grounding_validation: required_for_factual_answers
failure:
  max_model_attempts: 2
  schema_failure: retry_then_refuse
  safety_failure: refuse
  sensitive_data_failure: block_and_audit
  grounding_failure: retry_then_abstain
  mandatory_validator_unavailable: fail_closed
logging:
  raw_prompt_logging: disabled
  raw_output_logging: disabled
  redact_sensitive_fields: true
```

This is an illustrative configuration. The actual schema, accepted
values, policy precedence, and runtime validation must be implemented
and tested. Do not allow a request to override mandatory policy
settings.

## 11. Observability and Audit

### 11.1 Metrics

Monitor at minimum: - Request volume, latency percentiles, and error
rate. - Model/provider timeout and throttling rate. - Schema-validation
failure rate. - Safety, privacy, grounding, and injection-check
outcomes. - Retry count and retry success rate. - Refusal and
human-review rates. - Token usage and estimated cost by
application/tenant. - Tool invocation outcomes and denied actions. -
Validator availability and policy configuration changes.

### 11.2 Alerts

Define alerts for: - Sudden increases in blocked requests or validation
failures. - Unexpected changes in refusal rates or output
distributions. - Repeated injection attempts or tool-authorisation
denials. - Sensitive-data detection events. - Disabled, bypassed, or
unhealthy mandatory validators. - Unexpected model, prompt, schema, or
policy version changes. - Abnormal cost, token usage, latency, or retry
loops.

### 11.3 Audit event example

``` json
{
  "timestamp": "2026-10-23T00:00:00Z",
  "correlation_id": "opaque-id",
  "application_id": "support-assistant",
  "policy_id": "customer-support-answer",
  "policy_version": "1.0",
  "schema_version": "customer-support-answer-v1",
  "model_deployment": "approved-deployment",
  "validator_results": {
    "schema": "pass",
    "business_rules": "pass",
    "safety": "pass",
    "privacy": "pass",
    "grounding": "pass"
  },
  "decision": "approved",
  "attempt_count": 1,
  "duration_ms": 940
}
```

Use real timestamps and deployment identifiers in production. The
example intentionally omits prompt and response content. Protect audit
records against unauthorised access and modification, and apply a
documented retention schedule.

## 12. Testing and Assurance Plan

### 12.1 Unit tests

-   Valid and invalid schema payloads.
-   Missing, null, extra, malformed, and incorrectly typed fields.
-   Boundary lengths and numeric limits.
-   Business invariants and cross-field constraints.
-   Redaction and transformation correctness.
-   Action-policy mapping for every failure category.

### 12.2 Security tests

-   Direct prompt injection and jailbreak attempts.
-   Indirect injection in retrieved documents, files, URLs, and tool
    results.
-   Cross-tenant retrieval and access-control bypass attempts.
-   Tool argument tampering and privilege escalation.
-   Sensitive-data exfiltration and secret disclosure.
-   Malicious markup, oversized payloads, malformed encodings, and
    parser edge cases.
-   Attempts to influence policy via user-supplied metadata or claimed
    administrator status.
-   Attempts to induce retries that create duplicate side effects.

### 12.3 Reliability tests

-   Model timeout, throttling, and provider outage.
-   Validator outage and policy-store unavailability.
-   Log pipeline outage and queue saturation.
-   Circuit-breaker operation and safe fallback behaviour.
-   Retry exhaustion and total deadline enforcement.
-   Load, concurrency, rate-limit, and cost-budget tests.

### 12.4 Quality tests

-   Factual accuracy against approved reference answers.
-   Citation validity and evidence support.
-   Relevant abstention when context is insufficient.
-   Bias and language coverage where relevant.
-   Regression tests for every known production failure.
-   Human review of a risk-based sample of outputs.

### 12.5 Acceptance criteria

A release should not be approved until: - Mandatory controls cannot be
bypassed through client parameters or prompts. - Invalid outputs are
never returned as approved. - Critical policy failures cannot be
overridden by a passing average score. - Retry loops are bounded by
attempt, time, and cost limits. - Tool operations enforce independent
server-side authorisation. - Logs are redacted and access controlled. -
Fail-closed behaviour is tested for mandatory security dependencies. -
Policy and schema versions are recorded for every decision. - Security,
quality, performance, and regression tests meet the application's
documented thresholds. - Residual risks have an accountable owner and
explicit acceptance.

## 13. Deployment and Operating Model

### Development

-   Use synthetic or appropriately sanitised data.
-   Test schemas, policies, prompt templates, and adversarial cases.
-   Use isolated model resources and non-production identities.

### Test / pre-production

-   Run regression, adversarial, performance, and resilience tests.
-   Verify telemetry, alerts, access controls, and fail-closed
    behaviour.
-   Compare policy changes against a fixed evaluation dataset.

### Production

-   Deploy approved, versioned policy bundles through a controlled
    pipeline.
-   Restrict who can modify prompts, schemas, model deployments, and
    thresholds.
-   Use canary or staged rollout for material changes.
-   Monitor drift and incident signals.
-   Maintain rollback capability to a previously approved configuration.
-   Reassess the risk when the model, data sources, tools, or intended
    use changes.

### Change management

Each change should identify the affected model, prompt, schema,
validators, policy, test results, approver, rollout plan, and rollback
plan. Emergency changes must be logged and retrospectively reviewed.

## 14. Implementation Work Breakdown

  -----------------------------------------------------------------------
  Workstream                          Deliverables
  ----------------------------------- -----------------------------------
  Architecture and risk               Data-flow diagram, trust
                                      boundaries, threat model, risk
                                      classification

  Contracts                           Versioned JSON Schema/Pydantic
                                      models and compatibility tests

  Orchestration                       Guard service, model adapter,
                                      deadlines, retries, decision engine

  Security                            Authentication, authorisation,
                                      prompt-injection defences, tool
                                      broker, secret management

  Validation                          Schema, business, safety, privacy,
                                      and grounding validators

  Data protection                     Classification, redaction, logging
                                      policy, retention and access
                                      controls

  Operations                          Metrics, dashboards, alerts,
                                      runbooks, incident procedures

  Assurance                           Unit, integration, adversarial,
                                      performance, and regression test
                                      suites

  Governance                          Policy ownership, approvals,
                                      versioning, exception process,
                                      residual-risk register
  -----------------------------------------------------------------------

## 15. Recommended Delivery Sequence

1.  **Define the use case and risk level.** Identify what the AI can
    influence and the consequence of an incorrect response.
2.  **Map data flows and trust boundaries.** Include prompts, retrieved
    content, model provider, tools, logs, and downstream systems.
3.  **Define the contract and mandatory controls.** Write schemas,
    invariants, access rules, and failure behaviour before
    implementation.
4.  **Build the minimum guard service.** Implement request validation,
    approved model invocation, output validation, bounded retries, and
    safe refusal.
5.  **Add identity-aware retrieval and tool controls.** Enforce
    authorisation outside the model.
6.  **Add privacy, safety, grounding, and audit capabilities.** Select
    controls based on documented risk.
7.  **Automate tests and policy checks in CI/CD.** Block releases when
    mandatory tests fail.
8.  **Run adversarial and resilience testing.** Verify both expected
    behaviour and safe failure.
9.  **Deploy progressively.** Use staged rollout, monitoring, and
    rollback.
10. **Operate and improve.** Review incidents, false
    positives/negatives, model changes, and policy effectiveness.

## 16. Decisions to Resolve Before Production

The architecture owner should document: - Which use cases and risk tiers
are in scope? - Which model providers and deployments are approved? -
What data classifications may be sent to each provider? - Which
validators are mandatory for each task? - What is the maximum acceptable
latency and cost per request? - What is the retry count and total
deadline? - Which failures cause refusal, human review, or service
unavailability? - Can the application invoke tools or perform external
side effects? - What human approvals are required for high-impact
actions? - What is the audit retention and access model? - Who owns
policy changes, risk acceptance, and incident response? - What evidence
is required for release approval?



## 18. Recommended Architecture Improvements

This section turns the reference design into a more implementable enterprise pattern. These recommendations are additions to the original guard-and-validate flow and should be adopted according to the use case's risk classification.

### 18.1 Make the three enforcement boundaries explicit

The architecture should separate the following boundaries. They may be deployed as separate services or modules, but their responsibilities and trust assumptions must remain distinct.

| Boundary | Required controls | Security outcome |
|---|---|---|
| **Boundary A — Input and access** | Authentication, request validation, data classification, user/tenant authorisation, retrieval access control, input safety checks, tool eligibility | Untrusted requests cannot gain access to data or capabilities merely by persuading the model |
| **Boundary B — Model and output** | Approved model adapter, prompt/template versioning, structured output, schema validation, business invariants, privacy/safety/grounding checks | Model output remains a candidate until mandatory checks pass |
| **Boundary C — Decision and execution** | Explicit decision engine, bounded recovery, independent tool authorisation, human approval where required, response filtering, audit event | Only approved output or independently authorised actions can cross into downstream systems |

**Implementation rule:** a successful check at one boundary does not imply that another boundary has passed. For example, valid JSON is not proof of authorisation, and a safety classifier pass is not proof that a tool call is permitted.

### 18.2 Add a risk-tier control profile

Not every application needs the same expensive controls. Establish a documented risk profile for each use case and map each tier to mandatory controls.

| Illustrative tier | Example use | Minimum enforcement profile |
|---|---|---|
| **Tier 1 — Low impact** | Drafting, formatting, low-risk internal summarisation | Authentication where applicable, input limits, output contract, basic safety checks, redacted telemetry |
| **Tier 2 — Business relevant** | Customer support, internal knowledge assistant, document extraction | Tier 1 plus identity-aware retrieval, sensitive-data controls, business rules, grounding/citation checks where applicable, bounded retries, regression tests |
| **Tier 3 — High impact or action-capable** | Changing records, sending external communications, privileged operations, decisions with material consequences | Tier 2 plus fine-grained action authorisation, least-privilege tool broker, idempotency, human approval where required, stronger audit integrity, adversarial testing, explicit fail-closed behaviour |

These tiers are an example enterprise classification, not a substitute for the organisation's formal risk framework. The risk owner must approve the tier and any deviations. A lower tier must not be used merely to avoid a mandatory control.

### 18.3 Separate model generation from tool execution

Do not allow the LLM to execute privileged actions directly. Use a tool broker or equivalent server-side enforcement component.

Recommended sequence:

1. The model proposes a named tool and structured arguments.
2. The application verifies that the tool is enabled for the use case.
3. The broker checks the authenticated user's permissions and the target resource's access policy.
4. The broker validates arguments, bounds, allowlists, and business invariants.
5. The system obtains human confirmation for actions that require it.
6. The broker executes with a dedicated least-privilege identity.
7. The system records an operation ID and idempotency key where appropriate.
8. The tool result is validated before it is shown to the user or returned to the model.

The model's text, a generated confirmation statement, or a claim that approval was obtained must never substitute for an actual approval record or server-side authorisation decision.

### 18.4 Define policy precedence and immutable mandatory controls

Use an explicit policy precedence model to prevent a user prompt, application setting, or model output from weakening required controls.

Recommended precedence, from highest to lowest:

1. Legal, regulatory, and organisation-wide mandatory controls.
2. Platform security baseline and approved risk controls.
3. Application/use-case policy.
4. Task-specific settings that are explicitly permitted by the application policy.
5. User preferences that do not conflict with higher-level policy.

A request must not be able to disable validators, raise its own privileges, change the trusted policy version, or alter the mandatory failure action. Policy bundles should be validated before activation, access-controlled, versioned, approved, and rollback-capable.

### 18.5 Improve the decision engine

Represent validation results as structured outcomes rather than a single aggregate score.

Each validator should return, at minimum:

- `validator_id` and `validator_version`
- `status`: `pass`, `fail`, `indeterminate`, or `error`
- `severity`: policy-defined severity
- `failure_code`: stable machine-readable reason
- `mandatory`: whether passing is required for this use case
- `evidence_ref`: a safe reference to supporting evidence, if applicable
- `duration_ms`: elapsed validation time

The decision engine should apply explicit precedence:

1. Any mandatory security failure blocks approval.
2. An authorisation failure always denies the operation.
3. An unavailable mandatory validator produces a fail-closed result for the protected operation.
4. A retry is allowed only for an explicitly retryable failure.
5. An indeterminate high-impact result goes to human review or refusal.
6. Approval is possible only when every mandatory control has passed.

Do not average validator scores to override a hard security failure. If probabilistic scores are used, define thresholds through documented evaluation, monitor false positives and false negatives, and retain deterministic gates for hard requirements.

### 18.6 Add explicit service interfaces

Define stable interfaces between the orchestrator and its components. The following are conceptual interface names; implementations may use REST, gRPC, internal library calls, or message-based patterns.

| Interface | Input | Output |
|---|---|---|
| `validate_request` | Request envelope and trusted identity context | Normalised request or typed rejection |
| `authorise_context` | Caller identity, resource identifiers, requested purpose | Permitted resource references or denial |
| `evaluate_input` | Normalised request and policy version | Structured input-risk result |
| `generate_candidate` | Approved prompt, authorised context, model parameters | Candidate output and provider metadata |
| `validate_output` | Candidate, output contract, context, policy | Parsed result plus validator outcomes |
| `decide_action` | Validator outcomes, risk tier, retry state | `approve`, `retry`, `transform`, `refuse`, `review`, or `fail_closed` |
| `authorise_tool_action` | Caller identity, tool, arguments, target resource | Allow/deny decision and constraints |
| `execute_tool` | Authorised action, idempotency key, bounded arguments | Validated tool result and operation ID |
| `write_audit_event` | Redacted decision metadata | Audit acknowledgement or controlled telemetry failure result |

Interfaces should have typed contracts, timeouts, authentication, input-size limits, and documented error semantics. Do not pass raw credentials or unrestricted internal objects between components.

### 18.7 Harden prompt and context assembly

- Use approved, versioned prompt templates with an identified owner.
- Keep policy instructions in trusted configuration rather than mixing them with retrieved content.
- Label user-provided and retrieved content as untrusted data.
- Enforce context-size limits and strip unsupported file types or active content where appropriate.
- Track source IDs and access decisions for retrieved context.
- Do not include documents that the caller is not authorised to read.
- Keep secrets, access tokens, internal system prompts, and unrelated personal data out of model context.
- Test template changes against a fixed set of normal, adversarial, and regression examples.
- Treat prompt delimiters as a clarity aid, not as a security boundary.

### 18.8 Strengthen deployment and supply-chain controls

Integrate guardrail policy and code into the secure software delivery lifecycle.

**Pull request / build checks**
- Unit and contract tests for schemas, policy precedence, and decision mapping.
- Static analysis, dependency and secret scanning.
- Prompt-template and policy-schema linting.
- Regression evaluation against an approved test corpus.
- Tests confirming mandatory controls cannot be disabled through request parameters.

**Pre-production**
- Adversarial prompt-injection and data-exfiltration tests.
- Cross-tenant and resource-authorisation tests.
- Model/provider timeout, throttling, and validator-outage tests.
- Performance, concurrency, and cost-budget tests.
- Review of telemetry redaction and access restrictions.

**Release**
- Approval of model, prompt, schema, and policy versions as a coherent release set.
- Signed or otherwise integrity-protected release artefacts where supported by the platform.
- Staged deployment and automated health checks.
- Documented rollback to the last known-good policy/model configuration.
- A release record linking approver, test evidence, version identifiers, and residual risks.

A model version change can alter behaviour even when application code has not changed. Treat material model or provider changes as changes requiring evaluation and controlled rollout.

### 18.9 Add operational resilience and service objectives

Define service-level objectives appropriate to the use case. At minimum, measure end-to-end latency, availability, validator availability, policy-decision latency, cost per request, retry rates, and safe-failure rates.

Recommended controls:

- End-to-end deadlines propagated to downstream calls.
- Separate timeouts for model invocation and each validator.
- Circuit breakers for repeatedly failing dependencies.
- Bounded concurrency and queue depth.
- Per-user, per-tenant, and global budgets.
- Backpressure and load shedding during overload.
- A documented safe response when optional dependencies fail.
- Fail-closed behaviour when mandatory security dependencies fail.
- Recovery procedures for policy-store, audit-pipeline, and model-provider outages.

Decide explicitly whether the service may return a previously approved static response during an outage. Such a response must not be presented as current, personalised, or evidence-based if those properties cannot be verified.

### 18.10 Define audit integrity and privacy requirements

Audit data should support investigation without becoming a secondary repository of sensitive prompts and responses.

- Use correlation IDs across the gateway, guard service, model adapter, validators, tools, and audit pipeline.
- Record policy/model/schema versions and the final decision.
- Redact secrets and sensitive fields before telemetry leaves the enforcement component.
- Restrict audit read, export, and administrative permissions.
- Protect records against unauthorised modification and deletion using platform-appropriate controls.
- Define retention, deletion, legal-hold, and access-review requirements.
- Monitor for logging failures and ensure the response policy for audit outages is documented.
- Avoid logging full model reasoning or hidden internal prompts.
- Use sampled or separately approved content capture only where justified by a defined purpose and risk assessment.

For high-risk actions, an audit event should link to the action/transaction ID and approval record without treating the model's own narrative as evidence of execution.

## 19. Architecture Decision Records

Record the following decisions before production approval. Each record should include the decision, alternatives considered, rationale, owner, date, and consequences.

| ADR | Decision required | Recommended starting position |
|---|---|---|
| ADR-001 | Guard service deployment boundary | Central reusable service or shared library with an explicit enforcement boundary; select based on isolation, latency, and operational ownership |
| ADR-002 | Supported models/providers | Allowlisted deployments with documented data-handling and operational requirements |
| ADR-003 | Output contracts | Versioned schema per task; reject unknown fields unless explicitly allowed |
| ADR-004 | Validator failure behaviour | Fail closed for mandatory controls; safe fallback only where approved |
| ADR-005 | Retry and repair | Bounded attempts and deadlines; revalidate all repaired output |
| ADR-006 | Tool execution | Separate broker, independent authorisation, idempotency, and approval for consequential actions |
| ADR-007 | Logging | Redacted metadata by default; content capture requires documented approval |
| ADR-008 | Risk tier and human review | Explicit risk owner, thresholds, escalation path, and review evidence |
| ADR-009 | Policy change management | Versioned, tested, approved, monitored, and rollback-capable policy bundles |
| ADR-010 | Provider/model change | Regression evaluation and controlled rollout for material changes |

## 20. Updated Production Readiness Checklist

Use this checklist as a release gate. Mark each item as **Pass**, **Fail**, **Not Applicable with rationale**, or **Evidence Required**.

### Architecture and governance
- [ ] Data-flow diagram identifies every trust boundary and data recipient.
- [ ] Use case has a documented risk tier and accountable risk owner.
- [ ] Model, prompt, schema, and policy versions are traceable.
- [ ] Mandatory controls cannot be disabled by user input or application parameters.
- [ ] Exceptions have an expiry date, compensating controls, owner, and approval.

### Identity, data and tools
- [ ] Authentication and tenant/resource authorisation are enforced server-side.
- [ ] Retrieval access controls run before content enters the model context.
- [ ] Tools are allowlisted and execute under least-privilege identities.
- [ ] High-impact actions have appropriate confirmation/approval and idempotency.
- [ ] Secrets and unnecessary sensitive data are excluded from prompts and logs.

### Validation and decisions
- [ ] Schema, type, size, and business-rule validation are deterministic where feasible.
- [ ] Safety, privacy, and grounding controls are enabled according to risk.
- [ ] Every mandatory validator has defined failure and timeout behaviour.
- [ ] Hard failures cannot be overridden by a score average.
- [ ] Retry, repair, transform, refusal, and review actions are explicitly mapped.
- [ ] Every repaired candidate is revalidated through the required pipeline.

### Resilience and operations
- [ ] Deadlines, quotas, rate limits, and bounded retries are configured.
- [ ] Circuit-breaker, backpressure, and dependency-outage behaviours are tested.
- [ ] Logs are redacted, access-controlled, retained, and protected against tampering.
- [ ] Dashboards and alerts cover safety failures, validator health, cost, and latency.
- [ ] Incident response and rollback procedures have named owners.

### Assurance
- [ ] Unit, integration, adversarial, privacy, and regression tests pass.
- [ ] Cross-tenant access and prompt-injection scenarios have been tested.
- [ ] Tool arguments and external side effects are independently authorised.
- [ ] Performance and cost remain within documented limits.
- [ ] Residual risks and acceptance evidence are approved before release.

## 21. Updated Final Recommendation

Adopt the original guard-and-validate loop, but do not implement it as a single output filter or rely on the LLM to enforce its own restrictions. Implement a reusable enterprise guardrail capability with three explicit enforcement boundaries: input/access control, model/output validation, and decision/action enforcement.

The recommended implementation order is:

1. Establish the use-case risk tier, threat model, data-flow map, and accountable owners.
2. Define versioned output contracts and mandatory policy rules.
3. Implement authentication, request validation, and identity-aware access to data.
4. Add the controlled model adapter and deterministic validation pipeline.
5. Add explicit decision precedence, bounded retries, safe refusal, and fail-closed behaviour.
6. Place all tools and consequential operations behind independent authorisation.
7. Implement redacted audit events, metrics, alerts, and resilience controls.
8. Automate contract, security, adversarial, regression, and outage testing in CI/CD.
9. Release progressively with approval evidence, rollback, and post-release monitoring.
10. Reassess controls whenever the model, prompt, tools, data sources, or intended use changes.

**Minimum production acceptance condition:** no candidate output or tool action is approved unless every mandatory control for that use case has passed. If a mandatory security control cannot establish a trustworthy result, the system must refuse, defer for review, or fail closed rather than silently bypassing the control.


## 17. Final Recommendation

Adopt the diagram's guard-and-validate loop as the core pattern, but
implement it as a layered policy-enforcement service rather than a
simple output filter.

The minimum production baseline is: authenticated requests, input
validation, least-privilege context and tools, versioned output
contracts, deterministic schema and business validation,
risk-appropriate safety/privacy checks, bounded recovery, explicit
refusal/fail-closed behaviour, redacted audit events, and adversarial
regression testing.

For high-impact workflows, add independent authorisation, idempotent
tool execution, human approval, and formal risk acceptance. The system
should never treat model-generated text as proof of identity,
permission, consent, or successful completion of an external action.
