# Factor I: Governed Operations — Extended Guidance 

## Desired Outcome

When this factor is fully implemented, every operation a probabilistic caller performs against a system of record passes through a dedicated architectural layer — the control plane — positioned between the caller and the backend systems it touches. The depth of that layer is calibrated to the consequence of the operations it mediates: lightweight for read-only discovery, substantial for financial transactions and regulated data. No protocol-level interface masquerades as enterprise architecture. The organization can answer, for any operation routed through the control plane: what was validated, what credentials were used, what the outcome was — because an architectural layer exists that *owns* these concerns. Factors II–VII describe what that layer does. Factor I establishes that it exists.

---

## Terms of Art

This framework introduces four terms that the rest of the document — and the rest of the framework — depends on. Two additional concepts — the protocol gap and the consequence spectrum — do important structural work throughout this document. They are not terms of art; they are architectural observations that follow from the four terms below. Both are introduced in § 2 (How This Works).

### The reasoning layer

**Definition.** The architectural home of the LLM — the probabilistic caller that interprets context, reasons about what should happen, and expresses intent. In most agentic systems today, this is the LLM or AI agent that decides which operation to request based on a user's natural-language input, a set of available tools, and its own generated reasoning. In this framework, the reasoning layer owns intent — and nothing beyond intent. It does not validate, does not hold credentials, does not execute operations against systems of record. That is the full extent of its ownership.

### The control plane

**Definition.** The architectural home of consequences. Deterministic infrastructure positioned between the reasoning layer and backend systems of record. It owns what happens after intent is expressed: validation against authoritative state, credential management, execution, compensation when operations fail, error handling, audit. It is not a library inside the agent's code. It is not a middleware plugin. It is a distinct architectural layer with its own deployment, its own credential store, and its own audit surface. It serves every caller type — probabilistic agents, human-driven UIs, A2A delegations, scheduled jobs — because its behavior is bound to the operation and the backend, not to the caller that initiated it.

### Probabilistic caller

**Definition.** An LLM or AI agent whose outputs are generated stochastically. Optimized for helpfulness over accuracy, which means its default response to ambiguity is to try harder, not to stop. The same intent may produce structurally different requests across invocations. A probabilistic caller cannot guarantee it will follow a protocol, persist a key, send identical parameters, or know that a previous request succeeded. This is the caller type that breaks traditional software architecture assumptions — and the reason the entire framework exists.

### Deterministic caller

**Definition.** Code, a scheduled job, a rules engine, a human following a UI workflow. Outputs are repeatable given the same inputs. Traditional API design patterns — caller-supplied idempotency keys, session-based auth, structured retry logic — were designed for this caller type and generally work well.

---

## How This Works

Every agentic system has two layers. The reasoning layer expresses intent. The control plane owns consequences. The question is what that ownership looks like architecturally — and how its depth is calibrated to what it protects.

### The architectural decision

The control plane sits between every caller — human, A2A agent, scheduled job — and every backend system of record. The protocol stack (MCP, A2A, agents.md) terminates at one side; backend APIs terminate at the other. No operation bypasses it.

For many teams, recognizing this as a layer versus scattered constraints unique to each agent implementation, may be a novel architectural paradigm. In practice, the designs are familiar: version control, CI/CD pipelines, code review gates, and deployment processes are a control plane for software development. Load balancers, API gateways, and service meshes are a control plane for network traffic. What the reasoning layer demands is a control plane for operations against enterprise systems of record.

### The protocol gap and the consequence spectrum

Two concepts organize how the control plane is scoped and calibrated.

**The protocol gap** is the space between what the agentic protocol stack provides — discovery, invocation interfaces, task delegation, capability advertisement — and what enterprise operations require: authentication, authorization, business-rule validation, transaction integrity, credential management, compliance, audit. This gap is by design, not by omission. The protocols are correct to leave these concerns out — it is mechanism/policy separation well-applied (§ 9 Lineage). The control plane is what the enterprise builds in that space.

**The consequence spectrum** determines how deep the control plane goes at each point. This is not a checklist. It is a calibration principle. The factor paper's four-paragraph escalation — catalog query, payment processing, clinical records, trade execution — is the exemplar. Each step on the spectrum adds control plane depth:

**Discovery and read-only operations.** Authentication, basic rate limiting, and schema design ensuring tool descriptions expose business-level identifiers rather than backend implementation details. The control plane's job at this tier is routing and access verification — confirming the caller is authorized, shaping what tools are visible to it, and ensuring the interface describes operations in terms the caller can reason about rather than database columns it should never see. The control plane is present but lightweight. (Factors III and IV address interface design and tool visibility in depth.)

**Low-consequence writes.** Add validation — precondition checks against the system of record's actual state, not the caller's representation of it — basic audit logging, and deduplication to ensure retries do not produce duplicate side effects. The control plane validates before executing. Compensation paths may be simple at this tier: cancel the record, reverse the entry. (Factor V addresses how deduplication works when the caller is probabilistic; Factor II addresses how mutations are mediated.)

**Regulated and high-value writes.** Add scoped credential management where the probabilistic caller never sees tokens — just-in-time provisioning of narrow, operation-specific credentials that expire in seconds. Add compensation logic designed alongside every mutation capability, so the system knows how to reverse or correct each operation before deploying it. Add structured error responses that tell the caller exactly what happened, whether to retry, and what to do next — rather than returning raw backend errors the caller must interpret. Add a full audit trail recording what was requested, validated, executed, and returned. The control plane mediates every aspect of the operation. (Factor IV addresses how credential scoping and tool access are bounded; Factors VI and VII address error contract design and audit record structure.)

**Cross-system orchestration.** Add transaction sequencing across multiple backends, with compensation ordering that knows which steps to reverse and in what order when a multi-step operation fails partway through. Add correlation tracking so the audit trail captures the full operation as a single logical unit, not as disconnected calls to different systems. The control plane coordinates. (Factor II addresses multi-step mutation orchestration; Factor VII addresses cross-system audit coherence.)

"The dial turns in one direction" — as systems gain consequence, control plane depth becomes non-negotiable. But the inverse is also important: at the low-consequence end, building the full stack is over-engineering. Calibration means *both* directions. A product catalog query burdened with compensation logic and multi-system coordination is not well-architected — it is badly calibrated.

### What the control plane is not

The control plane is a new term applied to a familiar architectural pattern, and teams reaching for analogies tend to land on two that are close enough to be misleading.

**Not an API gateway.** An API gateway may be a component inside the control plane; it is not the control plane. API gateways operate at the HTTP/transport layer with request-response semantics on a single hop. The control plane operates at the semantic/intent layer, with potentially multi-step orchestration across multiple backends. An API gateway handles rate limiting and basic auth. It does not handle mutation validation against authoritative state, compensation when a multi-step operation fails partway through, or structured error responses designed for a probabilistic caller.

**Not a new ESB.** The control plane does not centralize business logic. Business logic stays in the systems of record where it belongs. The control plane's job is mediation at the boundary between the reasoning layer and those systems — validation, credential management, compensation orchestration, error handling, audit.

---

## When This Applies (and When It Doesn't)

The control plane is not always necessary. The deciding question is whether a probabilistic caller is operating against systems whose consequences matter — and how much those consequences matter determines how much control plane you build.

### Applies strongly

**Any probabilistic caller operating against enterprise systems of record.** This is the framework's primary context. If a probabilistic caller creates, reads, updates, or deletes data in a system that another business process depends on, a control plane is not optional. The protocol gap reasserts itself every time a probabilistic caller touches something that matters.

**Any multi-caller deployment.** When multiple callers — probabilistic and deterministic, agents and humans and scheduled jobs — can initiate operations against the same backends, the control plane is the coordination surface. Without it, each caller manages its own validation, credentials, and error handling independently — and the organization cannot enforce consistent behavior across the callers that share the same systems of record.

**Any regulated context.** PCI, HIPAA, SOX, GDPR — if a compliance framework governs the data the operation touches, the control plane is where compliance requirements are satisfied architecturally. The control plane provides the architectural surface where that evidence is produced.

### Applies partially

**Prototypes and non-production exploration.** A developer wiring an LLM to an API for exploration does not need a fully calibrated control plane. But the moment that prototype handles real data or faces real users, the protocol gap reasserts itself. The risk is the prototype-to-production path: the architectural decision to *not* have a control plane hardens as code accumulates. Teams that skip the control plane for speed in week one discover in month three that they have built a tightly coupled system where every agent implementation solves consequence management independently — and reversing that decision requires rearchitecting everything the agents touch.

### Does not apply

**Operations within the reasoning layer.** When operations stay inside the reasoning layer — context window, scratch space, working memory — there is nothing for a control plane to mediate. This framework addresses what happens when the reasoning layer crosses the boundary to touch external systems. What happens inside the reasoning layer is out of scope; other frameworks address that problem.

---

## Diagnostic Framework

Factor I asks the foundational questions: is there a control plane? Does it hold the right scope? Subsequent factors assume a control plane exists and interrogate what it does in depth.

The diagnostic is organized around the four terms of art defined above. Each term carries a concrete test. The sequence is ordered by where teams can start today — from what is most immediately observable and diffable against existing infrastructure, to what requires deeper architectural and organizational analysis.

### Caller type recognition: do you know what's calling your systems?

Most teams have infrastructure they can examine. The first diagnostic question is whether that infrastructure recognizes the difference between probabilistic and deterministic callers.

**Probabilistic caller test.** For each caller in your system, ask: does any part of the architecture assume this caller will behave deterministically? Four assumptions that hold for deterministic callers and break for probabilistic ones:

1. *Key persistence.* Does the architecture assume the caller will generate a unique idempotency key per logical operation and persist it across retries? A probabilistic caller fabricates syntactically valid keys that bind to no prior operation.
2. *Parameter consistency.* Does the architecture assume the caller will send identical parameters when retrying the same operation? A probabilistic caller retries through fresh generation — the intent is the same, the parameters may be different.
3. *Protocol adherence.* Does the architecture assume the caller will follow a specified retry protocol — back off, respect rate limits, stop after N attempts? A probabilistic caller's default response to failure is to try harder, not to stop.
4. *State awareness.* Does the architecture assume the caller knows what it did previously — that it can distinguish a retry from a new request, that it knows which operations have completed? A probabilistic caller has no guaranteed memory across invocations.

Any yes answer means the architecture is treating a probabilistic caller as if it were deterministic. The failure will be silent — the system works until the assumption is violated, and the violation is intermittent because the caller *sometimes* behaves deterministically by chance.

**Deterministic caller test.** Deterministic behavior in an agentic architecture means two different things depending on which side of the boundary you're examining — and teams that haven't drawn the boundary yet will have components where the two are conflated. A deterministic *caller* is deterministic in what it sends: given the same inputs, it produces the same request. A deterministic *control plane* is deterministic in what it does: given the same request, it applies the same validation, the same credential scoping, the same compensation paths. The difference is role — one initiates operations, the other mediates them.

1. *Caller or control plane?* For each component classified as a "deterministic caller," ask whether it is actually a caller at all. If a component in the path is deterministic *and* owns consequence-bearing responsibilities — it validates against authoritative state, it manages credentials, it handles compensation — it may be an unrecognized control plane component rather than a caller. The question is not "is this deterministic?" but "is this deterministic behavior the behavior of a caller, or the behavior of mediation infrastructure that hasn't been recognized as a control plane?"
2. *Does the execution path stay deterministic?* In modern systems, a single operation's path may cross between the control plane and the reasoning layer at multiple points — and anywhere the path touches the reasoning layer, that segment is probabilistic. A scheduled job that calls a deterministic API is deterministic. A scheduled job that asks an LLM to decide which API to call is not — the job is deterministic, but the caller that reaches the backend is probabilistic. Trace the path. Identify every point where the execution crosses into the reasoning layer. Each crossing changes the caller type for that segment, and the control plane must handle it accordingly.

### Control plane: does the mediation layer exist?

Once you understand what callers you have and how your infrastructure handles them, the next question is whether a distinct mediation layer sits between those callers and your backend systems. For a representative operation in your system, answer six questions:

1. *Does it exist as an architectural layer?* Is there a distinct component — with its own deployment, its own credential store, its own audit surface — between every caller and the backend systems of record? Or is consequence management scattered across agent code, MCP server handlers, and protocol-layer artifacts? The control plane is not a function library the agent imports. It is not logic embedded in MCP server handlers. It is a layer.
2. *Does it validate independently?* When a mutation request arrives, does the control plane check it against the system of record's actual state — or does it trust the caller's representation? If the agent says "this reservation is cancellable" and the control plane acts on that assertion without checking, it is relaying, not mediating.
3. *Does it own credentials?* Are backend credentials held by the control plane and invisible to the reasoning layer — or does any caller see, present, or manage a credential? If the agent passes an API key as a tool parameter, the control plane does not own credentials.
4. *Does it have compensation paths?* For each mutation the control plane can execute, does a designed compensation path exist — deployed alongside the forward operation, not improvised by the reasoning layer after a failure? If the system depends on the agent figuring out how to reverse a failed operation, the control plane does not own compensation.
5. *Is it caller-agnostic?* Does the infrastructure assume competencies from callers that probabilistic callers cannot reliably provide — key persistence, parameter consistency, protocol adherence, state awareness? If the trust boundary of operations assumes that every caller can do these things, the infrastructure is not caller-agnostic. The control plane's behavior should be bound to the operation and the backend, not to assumptions about caller competency.
6. *Is its depth calibrated to consequence?* Does the control plane's depth match the consequence of the operations it mediates — lightweight for a catalog query, substantial for a payment transaction, comprehensive for a multi-system trade execution? Or does it apply uniform depth across operations with materially different consequence profiles?

A control plane that fails any of the first five is either absent, incomplete, or not yet a control plane — it is infrastructure that *could become* a control plane once the gaps close. A control plane that fails the sixth exists but is miscalibrated — over-engineering low-consequence operations, under-protecting high-consequence ones, or both.

### Reasoning layer scope: are the right responsibilities on the right side?

This is the hardest assessment — and it cannot be completed through technical analysis alone. Understanding whether responsibilities are correctly placed requires both the findings from the caller type and control plane tests above, and a series of non-technical conversations: reviews of design assumptions, examination of organizational bias toward agent autonomy, and honest evaluation of use case scoping.

List every responsibility assigned to the reasoning layer in your architecture. If anything beyond "express intent" appears on the list, the architecture has misplaced a responsibility. Five responsibilities that do not belong to the reasoning layer:

1. *Validation.* Does the reasoning layer check whether its request is valid against the system of record's actual state — or does it express intent and let the control plane validate? If the agent verifies preconditions from its own context rather than relying on the control plane to check authoritative state, the architecture treats the reasoning layer as a validator.
2. *Credential management.* Does the reasoning layer see, hold, present, or manage any credential — API key, OAuth token, connection string, service account? If any credential is visible to the reasoning layer, the architecture treats it as a credential manager.
3. *Execution sequencing.* Does the reasoning layer coordinate multi-step operations — deciding which backend call comes next, passing system identifiers between steps, managing partial completion? If the agent orchestrates a saga, the architecture treats the reasoning layer as a workflow engine.
4. *Compensation.* Does the reasoning layer decide what to do when an operation fails partway through — which steps to reverse, in what order? If error recovery depends on the agent's judgment rather than designed compensation paths, the architecture treats the reasoning layer as a recovery coordinator.
5. *Audit production.* Does the reasoning layer produce the record of what happened — or does the control plane produce it as a byproduct of executing the operation? If the audit trail depends on the agent reporting what it did, the architecture treats the reasoning layer as its own auditor.

The test is not "could these responsibilities survive a misbehaving caller." The test is "does the caller have these responsibilities at all." Any yes answer is a misplacement.

### Reading the results

The tests produce binary findings — each responsibility, question, or assumption either passes or fails. But the findings have different architectural implications, and the progression from caller types to control plane to reasoning layer scope reflects increasing depth of analysis required:

Caller type failures indicate that traditional assumptions are being applied to a new caller type, or that callers are misclassified. These are the findings teams can act on fastest — they are visible in existing infrastructure and code. The architecture may work today, but the failures surface as intermittent production incidents when probabilistic callers violate the assumptions the system makes about them.

Control plane existence failures (questions 1–5) indicate that the mediation layer is absent or incomplete. Discovering this typically follows from the caller type analysis — once a team sees that it has probabilistic callers operating under deterministic assumptions, the question of what should mediate those callers becomes concrete. Control plane calibration failure (question 6) indicates the layer exists but the dial is set incorrectly.

Reasoning layer scope failures are the deepest findings. They often require the caller type and control plane analyses as prerequisites — a team cannot assess whether the reasoning layer is doing too much until it understands what its callers demand and what the control plane should own. These findings also require organizational examination: is the team biased toward pushing the wrong responsibilities to the reasoning layer? Are use cases scoped appropriately? The five misplaced responsibilities provide the checklist, but reaching the right answers requires more than a code review.

---

## Named Anti-Patterns

Four anti-patterns, ordered from what teams encounter first to what requires the deepest analysis to recognize.

### The Brittle Caller Assumption

*Diagnostic anchor: Probabilistic caller — the four broken assumptions; deterministic caller — the execution path test.*

The infrastructure assumes that every caller is competent to handle responsibilities that probabilistic callers cannot reliably perform. This is the most common anti-pattern because it is the industry's default state. Most enterprise infrastructure was built before probabilistic callers existed — its trust boundary places responsibilities like key generation, parameter consistency, retry discipline, and state tracking on the caller side because every caller *could* handle them. That assumption held universally until it didn't.

The result: the trust boundary is in the wrong place. The infrastructure trusts callers to supply idempotency keys, to retry with identical parameters, to follow backoff protocols, to know what they've already done. Probabilistic callers cannot guarantee any of these. The system works when the probabilistic caller happens to behave deterministically — and fails when it doesn't. The failures are intermittent, which makes them harder to diagnose than a clean break.

**Look for:** Infrastructure that requires the caller to supply idempotency keys, manage session state, or follow a specified retry protocol. Operations where the system's correctness depends on the caller behaving consistently across invocations.

**Ask:** "If a caller sent a structurally different request for the same logical operation, would our system recognize it as the same operation?" "Does our system assume callers will adjust their behavior based on error codes or retry signals?"

**Fix:** The trust boundary must move. Responsibilities that traditional infrastructure places on the caller — deduplication, retry safety, credential management, state tracking — must move to the control plane when probabilistic callers are in the picture. This is the paradigm shift the framework introduces, and it is addressed in depth across the subsequent factors: Factor II (Deterministic Mutations) addresses how mutations are mediated so the caller does not need to manage execution. Factor V (Safe Retries) addresses how retry safety is owned by the infrastructure rather than the caller. Factor VI (Recovery Contracts) addresses how error handling is structured so the caller does not need to coordinate recovery.

### The Overloaded Protocol

*Diagnostic anchor: Control plane — question 1 (does it exist as an architectural layer?).*

The team has loaded enterprise responsibilities onto protocol-layer artifacts — and this is almost always the path of least resistance. MCP servers are the first thing teams build. They're where tool definitions live. It is natural to put validation, credential management, and authorization logic in the same place the tools are defined. The result: tool descriptions encode business rules and validation constraints, MCP server handlers manage backend credentials directly, and authorization decisions get embedded in protocol-specific message handling. The protocols are no longer operating within their designed scope — they have been pressed into service as an enterprise mediation layer they were not built to be.

This works until it doesn't. A single change in business rules requires a series of changes across multiple protocol handlers. Adding a new protocol alongside MCP requires duplicating all the enterprise logic embedded in the servers. The day-to-day reality for teams is that "enterprise architecture" is actually a set of protocol-specific implementations with no shared mediation layer underneath. And critically: these implementations cannot support the consequence spectrum. Building enterprise controls into the protocol layer creates a crude cudgel, not a fine-grained instrument. Consequence-proportional depth requires an architectural layer designed for calibration, not a transport layer pressed into service as one.

**Look for:** Business-rule validation encoded in tool descriptions. Credential management inside MCP server handlers. Authorization logic or branching based on protocol-specific message structure. Any enterprise responsibility that would need to be reimplemented if a second protocol were added.

**Ask:** "Do our MCP server handlers manage credentials, enforce business rules, or make authorization decisions?"

**Fix:** Extract consequence-bearing responsibilities from protocol-layer artifacts into the control plane. The protocol layer should do what protocols are designed to do — discovery, invocation, task delegation. Enterprise mediation belongs in a layer that is protocol-independent. Factor II (Deterministic Mutations) addresses how mutation operations are routed through the control plane rather than handled inside protocol endpoints. Factor III (Intent-Based Communication) addresses how to design the interfaces between protocol-layer callers and the control plane. Factor IV (Bounded Access) addresses how tool visibility and credential scoping are managed at the control plane rather than encoded in tool schemas.

### The Consequence Mismatch

*Diagnostic anchor: Control plane — question 6 (is its depth calibrated to consequence?).*

The control plane exists — but it applies uniform depth across operations with materially different consequence profiles. A probabilistic caller has the same level of access and audit coverage whether it is querying a product catalog or accessing HIPAA-regulated patient records. Or the inverse: every operation is burdened with maximum-depth controls, making low-consequence operations unnecessarily slow and complex. Both directions are failures of calibration.

This anti-pattern often emerges after the first two are addressed. The team has recognized its caller types, built a control plane, extracted enterprise logic from protocol artifacts — and then applied the same depth everywhere because differentiating felt like premature optimization. It is not. The consequence spectrum (§ 2) exists precisely because uniform depth is wrong in both directions: it over-engineers the low end and under-protects the high end.

**Look for:** High-consequence data accessible through the same control plane path as low-consequence data with no differentiation in architectural depth. Regulatory or operational risk requirements documented but not mapped to specific control plane configurations. Low-consequence operations that are slow because they pass through controls designed for high-consequence ones.

**Ask:** "Can we show an auditor which controls apply to which operations — and do those controls match the impact of the operations they govern?"

**Fix:** Map operations to their consequence profile and calibrate control plane depth accordingly. The consequence spectrum provides the calibration tiers. Factor II (Deterministic Mutations) addresses how mutation mediation scales across consequence tiers. Factor V (Safe Retries) addresses how deduplication depth scales — low-consequence operations may need simple parameter matching; high-value operations need semantic deduplication. Factor VI (Recovery Contracts) addresses how error contract depth scales — a catalog query can return a simple error; a failed payment needs structured recovery guidance. Factor VII (Structural Observability) addresses how audit depth scales across the spectrum.

### The Reasoning-Layer Dependency

*Diagnostic anchor: Reasoning layer — the five misplaced responsibilities.*

This anti-pattern is driven by a fundamental bias in the industry: the assumption that the ideal end state is pushing as many responsibilities as possible to the reasoning layer. More autonomous is more mature. The agent that "handles everything" is the goal. This bias conflates autonomy with scope. An autonomous agent that operates within well-defined architectural boundaries — expressing sophisticated intent while the control plane owns consequences — is a well-designed system. An autonomous agent that manages its own credentials, validates its own preconditions, and coordinates its own recovery is not more capable — it is more exposed. This framework's position is clear: the health of an agentic system is measured by scoping the reasoning layer's responsibility to what it is actually capable of doing well within the enterprise. That scope is intent. Period.

When this anti-pattern is present, the reasoning layer has been given responsibilities beyond intent. The agent "handles its own auth." The LLM "decides whether to retry." Credential scoping relies on the agent requesting the right scope. Validation relies on the agent providing accurate preconditions. The architecture treats the probabilistic caller as an owner of consequence-bearing responsibilities rather than as an expresser of intent. Each of these works most of the time — and fails unpredictably, because the reasoning layer was never architecturally equipped to own them.

**Look for:** Auth flows that require the probabilistic caller to present or manage tokens. Retry logic that depends on the caller recognizing failure. Validation that trusts the caller's representation of current state.

**Ask:** "If we listed every responsibility assigned to the reasoning layer, would anything beyond 'express intent' appear?"

**Fix:** The ownership inversion — moving every responsibility beyond intent expression into the control plane — is the central argument of this framework. The subsequent factors address how: Factor II (Deterministic Mutations) addresses mutation mediation, Factor V (Safe Retries) addresses retry safety, Factor VI (Recovery Contracts) addresses error handling, and Factor VII (Structural Observability) addresses the audit trail that verifies the inversion is working. The reasoning layer should be powerful at what it does well — interpreting context, reasoning about intent, selecting operations. It should not be burdened with what it cannot reliably do.

---

## Industry Examples

The consequence spectrum is an abstract calibration principle until you see it operating within a single enterprise. These two examples show what happens when different operations in the same organization sit at different points on the spectrum — and what that means for control plane depth.

### Financial services: payment processing and trade execution

A financial services firm operates at two distinct points on the consequence spectrum. At the payment processing tier, an agent initiating a Stripe charge needs the control plane to manage OAuth credentials (the agent never sees API keys), validate the charge against account state (is the balance sufficient? is the account in good standing?), enforce idempotency at the boundary, and produce an audit record. The control plane is substantive but contained — one backend, one credential scope, one compensation path (refund).

Move up the spectrum to trade execution across a brokerage API, Salesforce CRM, and an ERP for settlement. The same enterprise, the same control plane — but now the depth changes materially. Transaction sequencing across three backends requires designed compensation at each step. Regulatory audit must capture the full chain — order, execution, allocation, settlement — as a single correlated sequence. Credential scoping differs per backend: the brokerage API requires trade-specific authorization, the CRM requires account-level access, the ERP requires settlement-tier credentials. And the consequence of failure is regulatory: a partially settled trade is not an engineering incident. It is a reporting event.

Same enterprise, same control plane, different calibration. The payment tier and the trade tier share the architectural layer but not the depth of mediation. This is the consequence spectrum in operation — and a financial services firm that applies uniform depth across both tiers is either over-engineering payments or under-protecting trades.

### Healthcare: clinical data access and order entry

A healthcare system illustrates the spectrum through the read/write boundary. Read access to patient records (Epic, Cerner) is governed — HIPAA requires audit trails and minimum-necessary access controls — but the control plane's depth is primarily access verification and audit production. The agent queries a patient's medication history; the control plane verifies authorization (is this clinician's agent scoped to this patient's records?), enforces minimum-necessary filtering (the agent sees medications relevant to the current encounter, not the full chart), and logs the access for compliance. No mutation, no compensation, no multi-system coordination.

Order entry changes everything. An agent placing a medication order touches pharmacy systems, billing, and clinical documentation. The control plane must validate against clinical decision support (drug interaction checks, allergy alerts — deterministic validation against authoritative clinical state, not the LLM's medical knowledge). Compensation paths must be designed: order cancellation, pharmacy notification, billing reversal. Multi-system coordination requires sequencing. And the consequence of failure is clinical: a duplicate medication order is not a billing inconvenience. It is a patient safety event.

The read path and the write path share the architectural layer. They do not share the control plane depth. A healthcare system that applies order-entry-grade controls to every medication history query buries clinicians in latency. A system that applies read-grade controls to medication orders exposes patients to clinical risk. The consequence spectrum calibrates both directions.

---

## Tradeoffs and Tensions

### Control plane depth vs. operational agility

A deeper control plane — validation, credential management, compensation logic, audit — adds latency, complexity, and engineering cost at every point in the stack. A thinner control plane increases operational risk. Both are legitimate concerns.

This is the bidirectional cost pattern. Too much depth over-constrains low-consequence operations: a product catalog query that passes through credential-management and compensation logic designed for payment transactions is not well-architected — it is slow, expensive to maintain, and signals to the team that the control plane is overhead rather than architecture. Too little depth under-protects high-consequence operations: a payment transaction with catalog-query-grade controls is an incident waiting to surface. The consequence spectrum is the resolution — but even calibrated depth has a cost at each point on the spectrum. Every validation check adds milliseconds. Every credential-scoping mechanism adds operational complexity. Every compensation path adds design and testing burden. The framework's claim is that this cost is justified by what it protects — but the cost is real, and teams should account for it.

### Organizational bottleneck risk

A dedicated control plane layer requires a team (or at least a capability) responsible for maintaining it. This creates organizational coupling: new backend integrations can only be added through the control plane, and the control plane team becomes a bottleneck. Feature teams wait for control plane support. Integration timelines lengthen. The pressure to bypass the control plane — "just connect the agent directly for now; we'll add the mediation layer later" — intensifies.

The mitigation is a clear scope boundary: the control plane mediates operations but does not own business logic. Backend teams own their domain; the control plane team owns the mediation layer. And the consequence spectrum provides a natural triage mechanism — not every integration needs the same control plane depth, so not every integration requires the same level of control plane team involvement. Low-consequence integrations can follow established patterns with minimal coordination; high-consequence integrations warrant the investment.

The tension is genuine. An organization that solves it well has a control plane team and capability that operates like a platform — providing self-service patterns for low-consequence integrations and hands-on collaboration for high-consequence ones. The control plane is infrastructure that is easy to build on, not a gate that is difficult to pass through. An organization that solves it poorly does not actually have a control plane — it has a cobbled-together set of integration-specific logic that someone has labelled as coherent architecture, and a central chokepoint that slows every project and incentivizes bypass.

---

## The Cost of Operating Without a Control Plane

When the control plane is absent, protocol-level interfaces masquerade as enterprise architecture. Authentication varies by integration. Credential management is per-agent, not per-operation. Validation logic is scattered across MCP servers, agent code, and backend middleware. Audit is fragmented or nonexistent. When an incident occurs — and in an agentic system operating against enterprise systems of record, incidents are a question of when, not whether — no single layer can answer "what happened and why." The investigation requires reconstructing the operation path across protocol-layer artifacts, agent code, and backend logs, none of which were designed to tell a coherent story.

The cost cascades. A new regulatory requirement arrives — say, the EU AI Act's high-risk provisions requiring that every agent action be logged, that human oversight be possible at structured intervention points, and that regulators can demand logs and technical documentation at any time. Without a control plane, every agent implementation that operates in a high-risk context must be individually audited, retrofitted with logging, and wired for intervention — across different frameworks, different backends, different teams. With a control plane, the logging and intervention capabilities are properties of the control plane itself, applied to every operation that crosses the boundary. The difference is not marginal. It is the difference between a regulatory response that takes a week and one that takes a quarter — and during that quarter, every agent operating without these capabilities is a compliance exposure.

The strongest version of the cost argument is not "something bad will happen." It is "you will not know that something bad happened." An agentic system without a control plane is not just unmediated — it is unobservable. No layer exists to produce audit records. No layer exists to intercept errors and return structured responses. No layer exists to deduplicate when the reasoning layer retries. No layer sits between the caller and the backend to validate before executing. The protocols connecting the reasoning layer to your systems of record will work perfectly. They were designed to. But a working protocol is not a production-ready architecture — and the distance between the two is the control plane.

---

## Lineage

The principle that the protocol should be thin — providing mechanisms, not dictating policies — is fifty-five years old. The agentic context is new. The combination changes what must be built in the space the protocols leave empty.

The intellectual arc begins with **Per Brinch Hansen's separation of mechanism and policy**, articulated in the RC 4000 system (1970) and elaborated by Levin, Cohen, Corwin, Pollack, and Wulf in the Hydra operating system (1975).[^1] The principle: a system should provide mechanisms for authorizing operations and allocating resources but should *not* dictate the policies governing those decisions. Policy belongs to the systems that own the consequence. Applied at the protocol layer: MCP, A2A, and agents.md provide mechanisms — discovery, invocation, task delegation, capability advertisement. They leave policies — authentication, authorization, business-rule validation, transaction integrity, credential management, compliance, audit — to the systems that own the consequence. The protocols are correct to do this. Factor I argues that this thinness is not a deficiency — it is Brinch Hansen's principle well-applied to a protocol stack designed for probabilistic callers. The protocols provide the mechanisms. The control plane is what the enterprise builds to own the policies — and the consequences — that the protocols correctly leave out.

The proximate structural template is **Adam Wiggins's Twelve-Factor App** (2011), specifically Factor IV: Backing Services.[^2] If the architectural shape feels familiar, this is why — the twelve-factor pattern of treating external systems as attached resources that the application does not own is the same posture the control plane takes toward backend systems of record. Wiggins treats databases, queues, caches, and API-accessible services uniformly as attached resources accessed via a URL and credential in configuration. The agentic context extends this template in three ways. Wiggins treats all backing services uniformly — a database and a third-party API follow the same attachment pattern. The control plane calibrates depth to consequence: a catalog query and a wire transfer require different architectural investment. Wiggins assumes a single first-party consumer — the twelve-factor app. The control plane serves multiple caller types — human, agent, A2A, scheduled job — under one regime. And Wiggins does not address the protocol gap because twelve-factor apps are not built on a multi-protocol stack. The agentic protocol stack is deliberately thin, and this is correct — Brinch Hansen's principle applied to protocols serving probabilistic callers. The principle is established. The context is new.

---

## Related Factors

This factor established the premise: the agentic protocol stack is deliberately thin, and the control plane is everything you build in the space the protocols leave empty. For how mutations enter the control plane and why the reasoning layer never connects directly to a system of record, see Factor II: Deterministic Mutations. For how the control plane communicates with callers through intent-aligned interfaces, see Factor III: Intent-Based Communication. For how the caller's tool namespace is scoped to prevent capability sprawl, see Factor IV: Bounded Access. For how retries are made safe when the caller is probabilistic, see Factor V: Safe Retries. For what the control plane returns when operations fail, see Factor VI: Recovery Contracts. For how every operation is recorded as a structured audit record, see Factor VII: Structural Observability.

---

## Notes

[^1]: Brinch Hansen, Per. "The Nucleus of a Multiprogramming System." *Communications of the ACM* 13, no. 4 (April 1970): 238–241. doi:10.1145/362258.362278. Supplementary: Levin, R., E. Cohen, W. Corwin, F. Pollack, and W. Wulf. "Policy/Mechanism Separation in Hydra." *Proceedings of the Fifth ACM Symposium on Operating Systems Principles (SOSP '75)* (1975): 132–140.

[^2]: Wiggins, Adam. *The Twelve-Factor App*, Factor IV: "Backing Services." 2011. https://12factor.net/backing-services