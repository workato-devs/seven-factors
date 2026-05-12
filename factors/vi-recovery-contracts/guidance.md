# Factor VI: Recovery Contracts — Extended Guidance

## Desired Outcome

When this factor is fully implemented, every capability invoked — success or failure — returns a structured result that tells the caller what happened, whether recovery is viable, and what to do next. The reasoning layer never infers outcome from unstructured text, never confabulates a recovery strategy from an ambiguous error, and never enters a generate-validate-fail loop because the error response left it guessing. The error channel that once carried free-text strings — an open injection surface for malicious services — now carries typed fields that constrain what can flow from backend to reasoning layer. The organization stops paying for confabulated recoveries, retry storms triggered by ambiguous errors, and the investigation time to trace agent misbehavior back to an unstructured error response.

---

## Key Concepts

This factor uses two of the framework's terms of art.

**Probabilistic caller.** A caller whose outputs are generated stochastically and whose default response to ambiguity is to try harder, not to stop. The probabilistic caller is optimized for helpfulness over accuracy — which means its default response to ambiguity is to try harder, not to stop. Every error handling tradition before this moment designed for a deterministic caller that executes branches the developer wrote, follows documentation, manages state predictably across calls, and handles errors through `if/else` against typed exceptions. Factor VI designs for the probabilistic one — a caller that interprets through inference, has no guaranteed memory across turns, and generates recovery strategies rather than executing them. The design spectrum, the diagnostic framework, and the anti-patterns in this document all follow from this distinction.

**Control plane.** The deterministic infrastructure layer between the reasoning layer and systems of record. In Factor VI's context, the control plane is where recovery contracts originate — the layer that transforms raw backend results into structured responses optimized for the probabilistic caller's next decision. The control plane owns the contract: it declares outcome, recoverability, and next action. The reasoning layer consumes the contract; it does not negotiate it.

---

## How This Works

The factor paper presents the recovery contract through one exemplar paragraph. This section goes deeper.

### The recovery contract

The industry uses "error handling," "error response," and "error message" interchangeably. Factor VI introduces a distinction that changes how teams design the return path from capability to caller.

A recovery contract is a structured response that declares outcome (success, failure, or partial success), classification (machine-readable category), recoverability (retryable, escalatable, or terminal), and next action (what the caller should do). The "contract" framing is deliberate: this is a commitment from the control plane to the probabilistic caller, not a convenience. The contract's fields are the interface; the specific values are the implementation. Changing the schema is a breaking change. Changing an error code is not. Most frameworks provide *error responses* — some information when something fails. Almost none provide *recovery contracts* — responses that satisfy all four properties.

### The design spectrum

Error response design today sits on a spectrum. The position you occupy determines recovery contract quality more than any specific field or schema — and the test that locates you on it is a single question: *what is the error response optimized for?*

**Context dump — optimized for nothing.** Give the probabilistic caller the full error context — journal state, execution history, raw messages — and let it reason about recovery. This is the default in most agent frameworks: errors become text in the conversation. The problem is that more context means more raw material for confabulation, not less. A deterministic caller executing coded branches can extract what it needs from a context dump. A probabilistic caller generating its next response treats every token as material for inference — and infers creatively when the signal is ambiguous.

**Developer-mediated structure — optimized for the deterministic caller.** Give the developer enough structured information to code the recovery logic. This is the durable execution model: typed error objects with fields for classification, retry control, and debugging payloads. The error model works well for its intended consumer — a workflow developer writing branching logic in source code. The problem for probabilistic callers: overloaded fields (arbitrary `details` payloads, `type` strings that serve both routing and observability) become noise when the consumer generates behavior rather than executing coded branches. The structure exists, but it is built for the wrong caller.

**Caller-decision-optimized fields — optimized for the correct response from a probabilistic caller.** Give the caller exactly the fields it needs to make a deterministic decision, and nothing else. Every field has one consumer and one purpose. The decision tree is deterministic: a boolean retryability field → wait and retry; an escalation flag → escalate to the operator; otherwise → fail fast. No reasoning required. The probabilistic caller does not need to interpret the fields — it needs to act on them. The fewer fields that require interpretation, the less room for confabulation.

Factor VI prescribes the third position. The design criterion: **every field in the recovery contract should answer one question for the caller, and no field should require the caller to reason about its contents.**

### The double duty principle

The design philosophy spectrum graduates *both* reliability and security along the same axis.

At the context dump position, the error channel is wide open. Every byte of error context flowing into the reasoning layer is material a malicious service could craft. Research on tool invocation prompt exploitation has demonstrated that tool return values — including error messages — are an active injection channel, successfully hijacking agent behavior across major frameworks and tools.[^1]

At the developer-mediated position, boolean and enum fields constrain part of the channel. But unbounded fields — `details` carrying arbitrary payloads, `message` carrying free-text — remain open channels of the same bandwidth as the raw context dump.

At the caller-optimized position, the channel is mostly constrained. A `retryable: boolean` cannot carry a payload. A `retry_after: integer` cannot carry a payload. An `owner_action_required: boolean` cannot carry a payload. An `error_category: enum` is constrained to a closed set. The only free-text exposure is a single purpose-defined prose field. Typed, validated fields do not just serve the caller's decision — they constrain what can flow from the backend into the reasoning layer.

Every field you type for the caller's sake is a field you close for the attacker's. The design philosophy that improves reliability — deterministic decision fields rather than context to reason about — simultaneously improves security — constrained injection surface rather than open channel. Retreating toward the context dump end for "richer context" or "more flexibility" degrades both simultaneously.

### The result envelope

What does a caller-optimized recovery contract look like in practice? The pattern is a small set of typed fields, each answering one question the probabilistic caller must resolve before generating its next action:

| Field | Type | Question it answers |
|---|---|---|
| retryability flag | boolean | Should I try again? |
| retry delay | integer (seconds) | When? |
| escalation flag | boolean | Should I escalate to the operator? |
| error category | enum (closed set) | What class of problem is this? |
| next-action guidance | prose | What is my next move? |

The specific schema is replaceable. The design philosophy is not: typed fields for the caller's decisions, with the only free-text exposure being a single purpose-defined prose field, not an unbounded payload. The strongest production implementation of this pattern — Cloudflare's RFC 9457 extension for AI agents — adds exactly six fields on top of an existing IETF standard and achieves a 98% reduction in token cost versus unstructured error pages.[^2] The elegance is in the constraint, not the comprehensiveness.

Most error handling treats outcomes as binary — success or failure. Multi-step operations can partially succeed: the charge went through but the room status update failed. GraphQL's response envelope, which returns both `data` and `errors` simultaneously, is the strongest precedent for treating partial success as a first-class outcome. A recovery contract that handles only binary outcomes forces the caller to guess about partial success — the most dangerous guessing of all, because the caller does not know which side effects have already occurred. The result envelope's outcome field must be a three-way discriminator — success, failure, partial success — not a binary flag.

### Where the industry sits today

The design spectrum is not theoretical — major frameworks occupy distinct positions on it, and the gaps are visible:

**OpenAI:** Context dump. No error flag. Errors returned as plain strings across the Chat Completions API, Responses API, and Agents SDK. The documentation states the format is up to the developer. **Anthropic:** First step toward structure — an `is_error` boolean on `tool_result` blocks separates "is this an error?" from "what happened?" but declares neither recoverability nor next action. Error details remain free-text in the `content` field. **MCP:** Same position — an `isError` boolean on `CallToolResult`, structurally identical to Anthropic's `is_error`. MCP is a protocol, not a vendor; it occupies the same point on the spectrum because it makes the same design choice. **AWS Bedrock:** Binary structure — `REPROMPT` (recoverable) vs. `FAILURE` (terminal) eliminates the retryability question but nothing else. **Temporal:** Developer-mediated structure — a five-field `ApplicationError` with mixed concerns. Comprehensive for the workflow developer, overloaded for the agent caller. **Inngest:** Closer to caller-optimized — three typed error classes (`NonRetriableError`, `RetryAfterError`, `StepError`), each answering one question. **Cloudflare RFC 9457:** Caller-decision-optimized — six fields, a deterministic decision tree, and a 98% reduction in token cost versus unstructured error pages.[^2]

### The stop condition handoff to Factor V

The recovery contract declares retryability. But a probabilistic caller optimized for helpfulness may retry despite a `retryable: false` signal — confabulating a workaround, reformulating through a different capability, or simply ignoring the signal. Factor VI owns the *signal*; Factor V's control plane owns the *enforcement*. The recovery contract makes stop conditions communicable; the control plane makes them consequential. This is the circuit breaker principle decomposed across the V/VI boundary: Factor VI provides the structured retryability signal that the circuit breaker reads; Factor V's control plane provides the structural stop when the caller ignores it.

---

## When This Applies (and When It Doesn't)

The principle — the caller should not have to guess at recovery — is nearly universal. The *shape* of the recovery contract varies by context.

**Mutations vs. reads: the shape varies, the principle does not.** Recovery contracts matter most visibly for mutations, where the caller's next action has irreversible consequences — retry means duplicate charge, escalate means lost sale, fail means abandoned workflow. Partial success is primarily a mutation concern. But reads that fail with ambiguous errors still cause confabulation, retry storms, and token burn — the same generate-validate-fail loops that cost $180 in twenty minutes.[^3] And reads that return 200 OK with garbage data — what Google's Factor XV calls "successful HTTP responses with useless or incorrect answers" — still need the caller to know something went wrong. If you have built the recovery contract infrastructure for mutations, extending it to reads is near-free. The honest guidance: do not carve out reads.

**A2A delegation: the protocol gap that governance must fill.** A2A provides transport for error information — a `TaskState` enum, `DataPart` schema, metadata, and extensions — but deliberately leaves error *semantics* to implementers. An A2A `TaskStatus` with `{state: FAILED, message: {parts: [{text: "Something went wrong"}]}}` is fully protocol-compliant. This is not an edge case — it is what the protocol permits by design. Organizations adopting A2A must build the recovery contract layer that the protocol omitted: error taxonomies via extensions, conventions for metadata keys, and machine-readable recovery suggestions alongside human-readable text. The protocol provides the transport. The enterprise must provide the contract. This is ultimately a governance decision — the organization choosing to enforce recovery contract standards that the protocol chose not to. Factor I in action.

**Where the guidance genuinely does not apply.** Pure prototyping environments with zero-consequence failures — where the cost of confabulation is genuinely zero and the developer is watching every interaction. Even here, the guidance is not harmful; it is just unnecessary. The boundary is narrow: the moment the prototype touches a system of record, handles real data, or operates without a human watching, the principle applies in full.

---

## Diagnostic Framework

Four dimensions, ordered from easiest to spot to hardest. Each maps to a named anti-pattern in the next section. Use these as a team walkthrough: trigger a failure and ask the questions in order.

**Dimension 1: Error flag presence.** *Does the response structurally distinguish success from failure?*

The baseline. If the response is a plain string or a 200 OK with domain-logic failure embedded in the body, the probabilistic caller must infer the outcome from text — and it will infer optimistically. The test: trigger a failure and examine what the reasoning layer receives. Is it a typed flag or a string to parse? One integration test.

**Dimension 2: Recoverability signal.** *Does the error response declare whether the caller should retry, escalate, or fail?*

Having an error flag without a recoverability signal is a half-measure. The probabilistic caller knows something went wrong but must infer what to do about it — and a caller optimized for helpfulness defaults to retry. The test: trigger three different kinds of failure — a transient timeout, a domain rule violation, a malformed request. Does the error response distinguish them? Can the probabilistic caller act differently on each without reasoning about the error content? A binary recoverable/terminal distinction is the minimum; typed retryability, escalation, and category fields are the target.

**Dimension 3: Next-action guidance.** *Does the error response tell the caller what to do, not just what went wrong?*

The difference between `OUTSTANDING_BALANCE` (classification) and "present the balance to the guest and ask for payment" (next action). Classification helps the control plane route; next-action guidance helps the probabilistic caller generate the appropriate response. This is harder to spot because teams that have classification believe they have solved the problem — but the gap between a category and an action is exactly where confabulation lives.

**Dimension 4: Caller-first design.** *Does the shape of the error response optimize for a probabilistic caller's next action — or for developer debugging?*

This is a design assumption, not a feature — and it is the hardest dimension to evaluate because teams with structured error handling may genuinely believe they have solved the problem. Their error responses *are* structured. But the fields serve the developer (`details`, `stack_trace`, `correlation_id`), not the caller (`retryable`, `suggested_action`, `escalation_required`). Both sets can coexist. If only the first set exists, you have built a debugging interface and called it a recovery contract.

Teams with structured error handling are most likely to miss this dimension because their responses *are* structured — for the deterministic consumer writing `if/else` against error types, not the probabilistic one generating its next action. The test: identify the fields in your error response that exist specifically to guide a probabilistic caller's next action — a flag, a recoverability signal, a next-action instruction. If they are present and the caller can act on them without wading through developer context, you have addressed this dimension. If they are buried in debugging payloads, stack traces, and serialized details, strip that noise away from the caller's channel. If they are missing entirely, build them — and route your developer signal to internal tracing where it belongs. The smell: the error response is comprehensive, but the probabilistic caller still has to reason about what to do.

**Maturity gradient:** Dimension 1 is table stakes — without it, the probabilistic caller cannot distinguish success from failure. Dimension 2 is necessary but not sufficient — without it, every failure triggers a retry. Dimension 3 is the functional standard — without it, the probabilistic caller confabulates the action from the category. Dimension 4 is the enterprise production standard — the error response serves the probabilistic caller's decisions through typed fields, and those same typed fields constrain the injection surface. Reliability and security close in the same architectural commitment.

---

## Named Anti-Patterns

Four anti-patterns, one per diagnostic dimension. Each names the control plane defect — the architectural smell the team should look for — not the agent behavior that results from it.

**The silent 200** *(Dimension 1: Clear failures).* The control plane returns a success status code for a domain-logic failure. The response says "OK" when the operation failed — and the probabilistic caller has no reason to doubt it. One practitioner documented an agent that "confidently mapped `company_revenue` to `employee_count`, invented values for fields that didn't exist in the source" — every API call returned 200 OK.[^3] The agent was not broken. The control plane lied about the outcome. Look for: any capability that returns success status codes for domain-logic failures. Fix: discriminated result type at the response level. The `isError` flag is the minimum, not the aspiration.

**The open retry** *(Dimension 2: Declared recoverability).* The control plane flags the error but does not declare whether recovery is viable — leaving the retry path open for the probabilistic caller. And it will retry, because it is optimized for helpfulness. Practitioner documentation of "doom loops" captures this precisely: when APIs return status codes without the response body detail, agents enter infinite retry loops; when the full structured error is surfaced, agents self-correct immediately.[^4] Look for: non-retryable failures that the caller retries anyway. If the caller cannot distinguish a transient timeout from a domain rule violation from a malformed request, the control plane has not closed the retry path. Fix: `retryable` boolean at minimum; `retry_after` + `owner_action_required` at target.

**Cause without action** *(Dimension 3: Actionable guidance).* The control plane classifies the failure — `OUTSTANDING_BALANCE` — but does not tell the caller what to do about it. The classification is machine-readable, well-structured, and completely silent on recovery. For well-known categories, capable models may infer the right action. For domain-specific or novel failures, the inference fails — and the gap between "what went wrong" and "what to do next" is where confabulation lives. Look for: error categories where the caller's next action requires domain reasoning to determine. If a domain practitioner would need to consult documentation to know the right recovery, the probabilistic caller will confabulate instead. Fix: a prose next-action field alongside the machine-readable category — the prose guides the reasoning layer, the category guides the orchestration layer.

**Built for the wrong caller** *(Dimension 4: Caller-first design).* The control plane returns a structured, typed, comprehensive error response — built for the deterministic caller. This is the hardest anti-pattern to recognize because the team has already invested in structured error handling and believes the problem is solved: `details` carries a serializable debugging payload, `type` serves both retry routing and dashboard labels, `message` is human-readable for log review. The fields are well-designed for a developer writing `if/else` against error types — but the consumer is a probabilistic caller generating its next action. Look for: error responses where the deterministic caller's fields are comprehensive but the probabilistic caller still has to reason about what to do next. Fix: build purpose-built caller-decision fields (flag, recoverability, next action) for the probabilistic caller, and route deterministic caller signal to internal tracing where it belongs.

---

## Industry Examples

### Hospitality: the Dewy Resort checkout error

A guest's checkout fails because of an outstanding balance. The same failure, three error response designs.

**Context dump.** The error channel returns: "Error processing checkout for booking BK-4821. Guest John Smith has an outstanding balance of $47.50 for minibar charges posted after the folio was closed. The PMS returned status code 402 with message 'Cannot complete checkout: open charges exist. Please review folio items added after 2026-04-27T14:00:00Z and ensure all charges are settled before attempting checkout.' Contact the front desk for assistance." The reasoning layer receives 89 tokens of context. It has everything it needs to *reason about* the problem — and everything it needs to confabulate a creative recovery.

**Developer-mediated structure.** The error response returns: `{type: "CHECKOUT_BLOCKED", message: "Outstanding balance prevents checkout", details: {balance: 47.50, currency: "USD", charges: [{type: "minibar", amount: 47.50, posted: "2026-04-27T16:30:00Z"}]}, non_retryable: true}`. The developer can code against `type` and inspect `details`. But the agent caller receives a debugging payload — the individual charge breakdown — alongside a single recovery signal (`non_retryable`). It knows not to retry. It does not know what to do instead.

**Caller-decision-optimized fields.** The recovery contract returns: `{retryable: false, owner_action_required: true, error_category: "outstanding_balance", what_you_should_do: "Present the outstanding balance of $47.50 to the guest and ask them to settle it before completing checkout.", retry_after: null}`. Three typed fields for three decisions. One prose field for the reasoning layer's next generation. The decision tree is deterministic: not retryable, owner action required, present the balance. No reasoning about charge breakdowns. No confabulation opportunity.

The domain shapes the contract: `error_category` values map to hospitality operations (outstanding balance, room not ready, guest not found), and `what_you_should_do` routes to the guest-facing interaction pattern appropriate to each failure. A checkout blocked by an outstanding balance requires the guest to act. A checkout blocked by a PMS timeout requires the agent to wait and retry. The recovery contract distinguishes these — the 200 OK does not.

### Healthcare: prescription failure recovery contracts

A clinical agent prescribes medication for a patient. Three fundamentally different failures demand three fundamentally different caller actions — and the domain raises the stakes on misclassification from financial cost to patient safety.

**Drug interaction detected.** `{retryable: false, owner_action_required: true, error_category: "clinical_contraindication", what_you_should_do: "Escalate to the prescribing physician. Do not suggest alternative medications. The interaction involves [drug A] and [drug B] — clinical judgment is required."}` The critical design choice: the next-action field routes to a *clinical* escalation path and explicitly prohibits the agent from suggesting alternatives. An agent that receives only `error_category: "clinical_contraindication"` without the next-action constraint might helpfully suggest a substitute — a confabulated clinical decision that no recovery contract should permit.

**Pharmacy system unavailable.** `{retryable: true, retry_after: 120, owner_action_required: false, error_category: "downstream_unavailable", what_you_should_do: "The pharmacy system is temporarily unavailable. Retry after the specified interval."}` A transient infrastructure failure. The contract says retry; the domain timing reflects pharmacy system recovery patterns.

**Prior authorization required.** `{retryable: false, owner_action_required: true, error_category: "authorization_required", what_you_should_do: "Route to the administrative authorization workflow. This is an insurance requirement, not a clinical issue — do not escalate to the physician."}` The critical design choice here: the next-action field routes to an *administrative* escalation path and explicitly distinguishes it from the clinical path. An agent that receives "authorization required" without routing guidance might escalate to the physician — wasting clinical time on an insurance process.

The domain shapes the contract in ways a generic schema cannot anticipate: error categories must be clinically meaningful, escalation paths must distinguish clinical from administrative workflows, and the consequence of misclassification is patient safety. The design criterion — every field answers one question for the caller — does not change. What changes is the *consequence* when a field is missing.

### Financial services: trade rejection recovery contracts

A trading agent submits an order that is rejected. Three failures, each with regulatory implications that shape the recovery contract's escalation paths.

**Insufficient margin.** `{retryable: false, owner_action_required: true, error_category: "margin_violation", what_you_should_do: "Route to risk management. Do not resubmit the order or adjust the quantity — margin assessment requires risk officer review."}` The agent must not attempt a smaller order as a workaround. The next-action field constrains the agent's generation space to a single path: risk escalation.

**Market closed.** `{retryable: true, retry_after: 43200, owner_action_required: false, error_category: "market_hours", what_you_should_do: "The market is closed. Retry after market open. No action required from the operator."}` The `retry_after` value carries domain-specific timing — market hours, not arbitrary backoff. The contract tells the agent exactly when to try again rather than leaving it to infer market schedules.

**Compliance hold.** `{retryable: false, owner_action_required: true, error_category: "compliance_hold", what_you_should_do: "Route to the compliance team with the hold reference. Do not resubmit, modify, or cancel — the order is under regulatory review."}` The critical constraint: the agent must not attempt *any* action on the held order. The next-action field is an explicit prohibition, not just a routing instruction. In financial services, the consequence of an agent acting on a compliance-held order is a regulatory event.

The domain shapes the contract: `error_category` values map to regulatory requirements (margin rules, market structure, compliance obligations), escalation paths must distinguish risk from compliance workflows, and the `what_you_should_do` field must include explicit prohibitions — things the agent must *not* do — alongside routing instructions. The cost of a missing or misleading recovery contract in financial services is not just token burn or customer frustration. It is a regulatory event with audit trail implications that connect directly to Factor VII's territory.

---

## The Cost of Ambiguous Errors

Ambiguous errors do not produce one failure — they produce cascading failures across token budgets, recovery accuracy, and security posture.

**Token cost.** One practitioner documented a $180 bill accumulated in twenty minutes when an agent entered a generate-validate-fail loop on malformed input — "it never occurred to the agent to stop and report the failure."[^3] An industry survey found that 92% of organizations implementing agentic AI reported costs higher than expected, with runaway retry loops as a primary driver.[^5] Cloudflare's evidence makes the economic case concrete: a 98% reduction in token cost (46,645 bytes to 970 bytes) when replacing unstructured error pages with RFC 9457 responses.[^2] The gap between "the model received a string" and "the model received a typed field" is measured in dollars per minute.

**Recovery accuracy cost.** Targeted, structured error feedback achieves a 26% relative improvement in task success rates compared to generic signals.[^6] Concrete, specific critique improves agent completion rates by nearly 30 percentage points over baseline; generic critique yields marginal gains.[^7] The gap between structured and unstructured error feedback is not incremental. It is the difference between an agent that recovers and an agent that confabulates.

**Security cost.** Research on tool invocation prompt exploitation demonstrated that crafted tool responses — including error messages — successfully hijacked agent behavior across Cursor, Claude Code, GPT-5, Claude Sonnet 4, Gemini 2.5 Pro, and Grok-4.[^1] Free-text error channels are active injection vectors. The security cost is not theoretical — it has been demonstrated against production tools. A `retryable: boolean` field cannot carry a prompt injection payload. Every untyped field in your error response is an open channel.

The recovery contract is not a convenience for the reasoning layer. It is the control plane's last opportunity to keep a probabilistic caller on a deterministic path — and off a costly one.

---

## Lineage

Factor VI draws from four traditions. The principle that the caller should not guess at recovery is old. The probabilistic caller is new.

Goodenough's 1975 POPL paper established that error handling is a design concern requiring structural support, not an afterthought handled by convention.[^8] Two lineages emerged from that foundation and both remain active. The exception lineage (PL/I → Ada → C++ → Java) encodes error information in throwable objects with type hierarchies. The Result/Either lineage (ML → Haskell → Rust) encodes the possibility of failure in the return type itself, where the caller literally *cannot* access the result without handling the error case. GraphQL's response envelope extends the Result tradition to treat partial success as a first-class outcome — both `data` and `errors` in the same response. All of these traditions assume the caller executes coded branches against the error structure. None anticipates a caller that generates its next action from the error content.

Standards-track formalization runs parallel. RFC 7807 (Nottingham and Wilde, 2016) defines a machine-readable format for error details in HTTP responses.[^9] RFC 9457 (2023) refines it with stronger extension guidance. Cloudflare's March 2026 extension bridges the gap to agent callers — adding the fields that eliminate reasoning (`retryable`, `owner_action_required`, `what_you_should_do`) on top of an established standard, with quantitative evidence of the result.[^2]

The durable execution tradition — Temporal, Inngest, Restate — validates structured error models in distributed systems at scale. These frameworks prove the infrastructure works. Their error models are not designed for the probabilistic caller's decision — but the machinery that delivers typed error objects to callers at production scale is directly reusable.

The circuit breaker tradition (Nygard 2007 → Netflix Hystrix → Resilience4j) established that when a caller cannot stop itself, an external mechanism must stop it.[^10] In the V/VI decomposition, circuit breakers become more accurate when recovery contracts provide structured retry signals, and less useful when they must infer retryability from status codes alone.

Independent convergence across 2024–2026 confirms the timing: structured error feedback improving agent recovery by 26%,[^6] doom loops resolving immediately when structured errors replace status codes,[^4] 79% of multi-agent failures traced to specification and coordination issues,[^11] and structured semantic signals being built for failure attribution after the fact — because they were not present during execution.[^12] Each team arrived independently at the same gap: error responses designed for deterministic callers do not serve probabilistic ones.

Factor VI's distinctive contribution is the double duty principle. Every antecedent tradition frames structured errors as serving recovery. Factor VI adds that the same typed fields that improve recovery simultaneously constrain the injection surface — making the error channel a security boundary, not just a reliability mechanism.

---

## Related Factors

This factor addresses what the control plane returns to the caller when operations complete — success or failure. For how mutations enter the control plane, see Factor II. For how the control plane enforces stop conditions when the caller ignores a non-retryable signal, see Factor V. For how intent-aligned interface composition reduces the frequency of errors that reach the caller, see Factor III. For how the control plane records every action including error responses — and the developer-privileged observability trap that is a shared concern — see Factor VII. For how governance scales with consequence, see Factor I.

---

## Notes

[^1]: Tool Invocation Prompt (TIP) exploitation research, arXiv 2509.05755. Researchers successfully exploited Cursor, Claude Code, GPT-5, Claude Sonnet 4, Gemini 2.5 Pro, and Grok-4 using crafted tool responses including error messages.

[^2]: Cloudflare, "Better error pages for AI agents using RFC 9457," Cloudflare Blog, March 11, 2026. https://blog.cloudflare.com/rfc-9457-agent-error-pages/

[^3]: Kevin Tan. Practitioner reports on agent error failures: "confidently mapped company_revenue to employee_count" (200 OK failures) and "$180 in tokens… twenty minutes… it never occurred to the agent to stop" (generate-validate-fail loop). blog.jztan.com.

[^4]: Pol Alvarez Vecino. "AI Agent Doom Loops." Documentation of agents entering infinite retry loops on HTTP 422 without response body detail, self-correcting immediately when structured errors are surfaced. medium.com/@pol.avec.

[^5]: IDC/DataRobot survey. 92% of organizations implementing agentic AI reported costs higher than expected.

[^6]: Zhu et al., "AgentDebug: Automated Debugging for LLM Agents," arXiv 2509.25370. Targeted, structured error feedback achieved a 26% relative improvement in task success rates.

[^7]: Atla-AI. Structured critique experiments: concrete critique improved completion rates by nearly 30 percentage points over baseline; generic critique yielded marginal gains. atla-ai.com.

[^8]: John B. Goodenough, "Structured Exception Handling," POPL '75: Proceedings of the 2nd ACM SIGACT-SIGPLAN Symposium on Principles of Programming Languages, pp. 204–224, ACM Press, 1975.

[^9]: Mark Nottingham and Erik Wilde, "Problem Details for HTTP APIs," RFC 7807, March 2016. Obsoleted by RFC 9457 (Nottingham, Wilde, and Dalal, July 2023).

[^10]: Michael Nygard, *Release It!: Design and Deploy Production-Ready Software*, Pragmatic Bookshelf, 2007. Circuit breaker pattern. Popularized by Martin Fowler (2014 bliki entry), implemented at scale by Netflix Hystrix (2012), succeeded by Resilience4j.

[^11]: Cemri et al., "MAST: Multi-Agent System Taxonomy," arXiv 2503.13657, ICLR 2025. 79% of multi-agent failures originate from specification and coordination issues.

[^12]: ErrorProbe, "Towards Self-Improving Error Diagnosis in Multi-Agent Systems," arXiv 2604.17658, April 2026.
