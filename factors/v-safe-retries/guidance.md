# Factor V: Safe Retries — Extended Guidance

## Desired Outcome

When this factor is fully implemented, no capability invoked against a system of record produces a duplicate side effect — regardless of how many times the reasoning layer re-expresses the same intent, regardless of whether it retries through the same capability or a different one, regardless of whether it knows it is retrying. The control plane deduplicates at the boundary between probabilistic and deterministic, using keys it derives or manages itself — because a probabilistic caller will fabricate syntactically valid idempotency tokens, retry with structurally different parameters, lose context and re-initiate operations it already completed, and never know the difference. When retrying is futile — the error is non-retryable, the condition is structural — the control plane enforces the stop rather than relying on the reasoning layer to respect the signal. The organization can verify, for any mutation: where the deduplication key originates, whether completed steps re-execute on retry, and whether non-retryable errors are enforced or merely communicated. Idempotency is a property of the control plane, not a protocol the caller follows.

---

## Key Concepts

This factor uses two of the framework's terms of art.

**Probabilistic caller.** A caller whose outputs are generated stochastically and whose default response to ambiguity is to try harder, not to stop. The probabilistic caller is optimized for helpfulness over accuracy — which means it will cheerfully participate in idempotency patterns, generating syntactically valid keys and passing them as parameters, but it cannot be trusted to do so correctly. It has no durable state between invocations. It cannot distinguish a retry from a new request. And when an idempotency token appears as a tool parameter, it fabricates one — generating a syntactically valid UUID that binds to no prior operation.

**Control plane.** The deterministic infrastructure layer between the reasoning layer and systems of record. In Factor V's context, the control plane is where deduplication happens — at the boundary where operations cross from probabilistic intent into deterministic execution. The control plane derives or manages deduplication keys, persists them, normalizes parameters, and detects duplicates without relying on caller cooperation.

---

## What Goes Wrong

A probabilistic caller produces duplicate side effects in five distinct ways. Each has a different structural cause and a different architectural fix — which is why teams that treat idempotency as a single concern miss the specific failure mode that costs them.

**The confused retry.** The reasoning layer retries the *intent* through a fresh probabilistic generation, producing a structurally different but semantically equivalent request. Parameter values vary. Tool selection may vary. The agent calling `create_order` after `reorder_last` timed out is not retrying the same request — it is retrying the same *goal* through a different path. Hash-based deduplication on exact parameters fails because the parameters are never exact.

**Context-loss re-initiation.** The reasoning layer loses context — token window overflow, session boundary, tool error that clears state — and re-initiates an operation it already completed, with no awareness that it is retrying. Multi-agent system studies have documented failure rates between 41% and 86.7% across execution traces, with conversation resets and re-executed tasks appearing consistently across frameworks.[^1] The agent is not retrying. It is starting over. Retry-aware logic does not help a caller that does not know it is retrying.

**Misinterpreted success/failure.** The operation succeeds but the reasoning layer interprets the response as failure (or vice versa) and retries. Factor VI's error contracts reduce the *frequency*. Factor V's deduplication makes the *consequence* safe regardless.

**Multi-agent duplicate initiation.** Two or more agents independently decide the same operation is needed. Neither is retrying — both believe they are initiating. Traditional per-caller deduplication misses this because each caller's request is genuinely novel from its own perspective.

**The saga-initiation problem.** A multi-step workflow fails mid-execution. Compensation runs. The reasoning layer re-initiates the entire workflow. The saga orchestrator correctly manages the new saga — but cannot deduplicate across saga initiations because each has a unique saga ID. One practitioner reported finding over 200 orphaned records a week after deployment, each one a customer who had received double billing notifications.[^2]

Practitioners who say "we have idempotency" typically mean they have implemented one pattern — usually caller-supplied keys — that addresses exact-parameter retries from a single caller.

---

## How This Works

The control plane is deterministic infrastructure. Deduplication happens at the boundary where operations cross from the probabilistic reasoning layer into the control plane — the point where intent becomes execution. Every capability invoked — whether a tool call, a CLI command, a code execution, or an agent-to-agent delegation, and whether from a human, an agent-to-agent caller, or a scheduled job — crosses this boundary. The architectural commitment is that the control plane prevents duplicate side effects without relying on the caller to signal "this is a retry."

### Layer 1: Deterministic key sourcing

The foundational commitment: the reasoning layer never originates an idempotency key value from its own probabilistic output.

Two variants satisfy this commitment depending on what the backend requires. When the backend handles deduplication internally and does not require a caller-supplied key, the control plane manages the key invisibly. The reasoning layer calls the mutation tool; the control plane derives or assigns a key behind the boundary; the backend never receives a duplicate. The reasoning layer is unaware that deduplication exists.

When the backend requires a caller-supplied idempotency key — as payment processors, many SaaS APIs, and systems following the Stripe/IETF pattern do — the control plane provides a deterministic generation step. The reasoning layer invokes this step to obtain a key, then passes that key as a parameter to the mutation tool. The key appears in the tool call, but its value was produced by deterministic infrastructure, not by the LLM's generation. The caller participates in the flow but does not originate the value. This variant matters because the Stripe-perpetuated pattern dominates production APIs: eliminating the key parameter is often not an option; ensuring its value comes from a deterministic source always is.

Both variants achieve the same outcome: no key value enters a backend system that was fabricated by probabilistic generation.

### Layer 2: Step-level replay prevention

Once an individual operation within a multi-step workflow completes, re-execution of that step is prevented. Three mechanisms achieve this, and they are not mutually exclusive.

Durable execution frameworks provide **journaled replay** — once a step completes and its result is recorded in the journal, re-execution of the workflow skips that step and returns the journaled result. **Backend-side cached responses** achieve the same outcome when the backend recognizes a duplicate key: the same idempotency key arriving twice returns the cached result rather than executing a second time. And **state-condition checking** — the simplest mechanism — uses the deterministic key from Layer 1 to check whether the operation's side effect already exists before attempting it. Does this booking reference already have a completed Stripe charge? Is this room already released in the PMS? The control plane checks the state, finds the operation already completed, and skips it. No journal, no cached response — just a lookup keyed on the deterministic identifier.

Layer 2 builds directly on Layer 1: every mechanism here is keyed on deterministically sourced identifiers.

### Stop condition enforcement

Deduplication prevents duplicate *side effects*. It does not prevent futile *retry loops*. A probabilistic caller that receives a non-retryable error — a checkout blocked by an outstanding balance, a prescription rejected by formulary rules — may still retry. The caller is optimized for helpfulness over accuracy; its default response to failure is to try harder, not to stop. The control plane deduplicates each attempt, preventing financial harm — but the caller is still burning tokens on a structurally impossible recovery.[^6]

The control plane should enforce retry caps informed by Factor VI's retryability signal. When an error contract declares `retryable: false`, the control plane enforces the stop structurally — not as a suggestion to the reasoning layer, but as a gate the reasoning layer cannot bypass. Error contracts make stop conditions communicable; stop enforcement makes error contracts consequential. Neither alone is sufficient.

---

## When This Applies (and When It Doesn't)

Two applicability axes scope Factor V's guidance.

**Consequence severity determines the investment.** Factor V's mechanisms are warranted only for operations that produce side effects — and the sophistication of those mechanisms should match the consequence of a duplicate. Reads do not produce side effects; retrieving the same data twice does not create duplicates, charge anyone, or corrupt state. If your agent only reads, Factor V's guarantees are not needed. (Factor VII addresses the distinct question of *auditing* reads in regulated domains, but that is an accountability concern, not a deduplication concern.) For low-consequence mutations in prototyping environments, idempotency may be deferred entirely. For production systems processing mutations against systems of record, deterministic key sourcing (Layer 1) and step-level replay prevention (Layer 2) are the baseline. Parameter canonicalization within those layers is a design choice — it adds resilience against same-tool parameter variance but introduces its own complexity, and teams should evaluate whether contextual-identifier keying already covers their high-consequence mutations before investing. The failure modes that escape Layers 1 and 2 — cross-tool intent duplication, cross-caller independent initiation — require cross-factor design work with Factor III (intent-aligned interface composition) and Factor IV (access scoping), not just more sophisticated deduplication. Factor I's consequence spectrum governs: the governance dial turns in one direction.

**Caller architecture determines the failure surface.** A single-agent system faces the confused retry, context-loss re-initiation, and misinterpreted success/failure — all of which are addressed by Factor V's own mechanisms (Layers 1 and 2) and Factor VI's error contracts. A multi-agent system additionally faces cross-caller independent initiation, where two agents independently decide the same operation is needed and neither is retrying. This failure mode cannot be solved by deduplication alone — it requires Factor IV's access scoping to prevent duplicate initiation paths. Teams running single-agent systems can focus on Factor V's owned dimensions. Multi-agent systems require access scoping from the start — bounding which agents can initiate which classes of mutation — because deduplication alone cannot distinguish a legitimate parallel initiation from an unwanted duplicate. Retrofitting access scoping after duplicate initiations surface in production is significantly harder than designing it in.

### Implementation considerations

**Canonicalization as a design choice.** Canonicalization is not a default recommendation — it is a design choice with real costs: domain-specific equivalence rules that must be maintained, a new bug surface where incorrect normalization silently corrupts deduplication, and ongoing complexity as the domain evolves. For most teams, contextual-identifier-based key derivation within Layers 1 and 2 handles the high-consequence failure modes without canonicalization. Reserve it for specific mutations where the consequence of a missed same-tool duplicate justifies the investment — and where contextual-identifier keying alone leaves a gap.

**Performance overhead.** Every mutation at Layer 1 requires a dedup store lookup or state check. At high throughput, the dedup store becomes a bottleneck. The consequence spectrum governs: high-consequence mutations justify the overhead; low-consequence mutations in high-throughput environments may accept lighter mechanisms.

**Fail-fast design.** The dedup check is cheap. The mutation it gates is expensive — both in compute and in consequence. Operations with high consequence or high reversal cost should be designed to fail fast at the deduplication boundary rather than execute and require compensation after the fact. The dedup store lookup that catches a duplicate before it reaches any backend system is orders of magnitude cheaper than the remediation that follows a duplicate that executes — whether that remediation is a customer refund, a regulatory filing, an infrastructure teardown, or an internal incident review explaining to executive stakeholders why an automated process did something expensive twice.

**Deduplication windows.** Every dedup store needs a retention policy. Time-windowed deduplication — rejecting operations that match within a defined window — is a pragmatic constraint on Layers 1 and 2, not a separate architecture. The tradeoff: too short and duplicates escape the window; too long and legitimate re-operations are captured by it. There is no universally correct window — the right duration depends on the operation's domain semantics and must be calibrated per mutation type.

---

## Diagnostic Framework

Five dimensions, ordered from easiest to spot to hardest to address. The first three are Factor V's own territory — mechanically achievable and testable. The last two are cross-factor and require design work beyond deduplication.

### Factor V dimensions

**Dimension 1: Key value sourcing.** *Can the reasoning layer originate an idempotency key value from its own output?*

Trace the value pipeline for every idempotency key in your mutation tools. If the reasoning layer can populate a key parameter with a self-generated value — a UUID it fabricated, a string it composed — the control plane does not own the key. The key may legitimately appear as a tool parameter (many backends require it), but the value must come from a deterministic source: a control-plane generation step, a derived hash, a journal entry, a state lookup. The diagnostic question is not "does the key appear as a parameter?" but "where does the value come from?" If the answer is "the LLM's output," every retry creates a new operation.

**Dimension 2: Step-level replay.** *When a multi-step operation retries, do completed steps re-execute?*

Trigger a retry of a multi-step workflow that partially completed. Does the system re-execute the steps that already succeeded, or does it skip them? If a checkout that succeeded in Stripe but failed at the PMS retries, does Stripe see a second charge attempt? The test: fail a multi-step operation mid-execution and observe whether completed steps produce duplicate side effects.

**Dimension 3: Stop condition enforcement.** *When a non-retryable error occurs, does the control plane enforce the stop — or does it rely on the reasoning layer to respect the signal?*

Trigger a non-retryable error and observe whether the reasoning layer can continue retrying the same intent through alternate paths. If it can, the stop enforcement has a gap. The test is not whether the error contract communicates retryability (that is Factor VI's diagnostic), but whether the control plane *enforces* the signal when the reasoning layer ignores it.

### Cross-factor dimensions

**Dimension 4: Interface composition (Factor III).** *Can the reasoning layer reach the same irreversible mutation through multiple interfaces in its namespace?*

Two interfaces that touch the same backend systems and produce the same side effect — but through different parameter schemas or different levels of identifier resolution — create a path that no deduplication mechanism can recognize as duplicate. The fix is intent-aligned interface design that collapses duplicate paths to the same backend operation, not better deduplication.

**Dimension 5: Initiation scoping (Factor IV).** *Can multiple callers independently initiate the same irreversible mutation?*

Content-hash deduplication faces an unsolvable ambiguity here: the same parameters might represent a legitimate duplicate (a customer genuinely ordering twice) or an unwanted duplicate (two agents processing the same need). The content alone cannot distinguish them. The fix is access scoping that prevents duplicate initiation paths.

**Workflow re-initiation** sits at the intersection of Dimensions 2 and 5. A multi-step workflow that fails and is re-initiated with a new saga ID cannot be deduplicated at the saga level — each initiation is unique. Workflow-level deduplication keyed on domain intent (`checkout:BK-4821` rather than `saga-run-7f3a`) can help, but it faces the same legitimate-vs-unwanted ambiguity: a guest who checks out, then returns to extend their stay and checks out again has two legitimate checkouts for the same booking. The domain context determines which re-initiations are duplicates, and that determination often cannot be made mechanically.

The failure mode that does not appear here as a standalone dimension — misinterpreted success/failure — is diagnosed via Factor VI's diagnostic framework. Factor V's deduplication makes the *consequence* safe regardless of error contract quality.

---

## Named Anti-Patterns

### Factor V anti-patterns

**The fabricated token** *(Dimension 1).* The idempotency key is a tool parameter — and the reasoning layer populates it from its own output. One practitioner described the LLM generating syntactically valid UUIDs "with complete conviction" — each one binding to no prior operation.[^4] Every retry is a new operation. Orphaned records surface as double billing notifications a week later. Fix: provide a control-plane generation step that produces the key value, or move key derivation behind the boundary entirely.

**The unreplayable workflow** *(Dimension 2).* A multi-step operation fails mid-execution and the system retries from the beginning, re-executing steps that already completed. The checkout charged the guest in Stripe, then failed at the PMS update. The retry charges the guest again. Fix: journaled replay, backend-side idempotency using deterministically sourced keys, or state-condition checking that verifies whether the step's side effect already exists before re-executing.

**The futile retry loop** *(Dimension 3).* A non-retryable error occurs and the reasoning layer retries anyway, reformulating the request or attempting alternate tool paths. Each attempt is deduplicated, preventing duplicate side effects, but the loop consumes tokens and may trigger confabulated workarounds that compound the problem. One practitioner reported $180 in tokens burned over twenty minutes on a structurally impossible recovery.[^6] The deduplication is working — the stop enforcement is not. Fix: the control plane enforces retry caps informed by Factor VI's retryability signal.

### Cross-factor anti-patterns

**The shadow path** *(Dimension 4).* Two interfaces in the namespace share backend surface but present differently — different parameter schemas, different levels of identifier resolution. Someone added `reorder_last` as a convenience without realizing it reaches the same mutation as `order_item`. The reasoning layer calls one after the other timed out — a confused retry through a route nobody designed as a duplicate. Fix: collapse duplicate-effect interfaces into a single intent-aligned interface (Factor III).

**The duplicate dispatch** *(Dimension 5).* Two agents are independently sent to do the same job. Each agent's request is novel within its own context — neither is retrying. The customer is charged twice. Fix: scope access so that only one agent can initiate a given class of irreversible mutation (Factor IV), or route all initiation paths through a shared control plane operation that can apply domain-context-aware deduplication.

---

## Industry Examples

Three domains illustrate how the same deduplication principle adapts to different backend systems, regulatory environments, and consequence severities.

### Hospitality: The Dewy Resort checkout

A guest checks out of Dewy Resort. The checkout touches three backend systems: Stripe processes the final charge, Salesforce updates the guest contact record, and the PMS releases the room. The Stripe charge succeeds. The PMS call times out. The reasoning layer retries the checkout — it cannot distinguish a timeout from a failure.

Without Factor V, the retry creates a second Stripe charge. The guest is billed twice. The duplicate surfaces when the guest disputes the charge or when a reconciliation job catches the mismatch — days later.

With Factor V, the reasoning layer obtained a deterministic token from the control plane before the first attempt (Layer 1). The control plane derived a stable token from the contextual identifier — the booking reference — and returned it: `idem-checkout-BK4821-001`. On retry, the same booking reference produces the same token. Stripe recognizes the duplicate key and returns the cached result (Layer 2). The control plane retries only the failed PMS step. The guest is charged once. The room is released. The key is a tool parameter. The key appears in the mutation call. But the key's *value* was produced by deterministic infrastructure, not by the LLM. The caller holds the key; the control plane owns it.

### Financial services: Trade deduplication across venues

A trading agent submits a buy order to a matching engine. The acknowledgment is delayed beyond the agent's timeout threshold. The agent re-submits — either as a confused retry with slightly different parameters, or as a context-loss re-initiation where it has lost awareness of the original submission.

The consequence spectrum is severe: a duplicate trade is a regulatory event. Cross-venue deduplication is the specific challenge — the same order routed through different execution venues produces structurally different submissions. The control plane must recognize intent identity across different order representations: same instrument, same quantity, same direction, same time window. The deduplication key is derived from domain semantics — the trade intent — not from the submission format of any individual venue.

### Healthcare: Prescription deduplication across pharmacy systems

A clinical agent prescribes medication for a patient. The EHR integration times out. The agent re-initiates, expressing the same clinical intent through a request with different surface characteristics — brand name instead of generic, a slightly different dosage format, a different pharmacy identifier.

The consequence of failure is patient safety: a duplicate prescription means the patient may receive double the intended medication. The control plane must canonicalize across pharmaceutical representations — matching brand name to generic equivalent, normalizing dosage formats, and recognizing that the same patient, same medication, and same clinical encounter constitute the same prescription intent regardless of how the parameters are expressed. In healthcare, the consequence severity demands the most rigorous canonicalization — and the regulatory environment (DEA Schedule II controls, state pharmacy board requirements) makes the audit trail from Factor VII inseparable from the deduplication guarantee.

---

## The Cost of Getting Deduplication Wrong

Deduplication can fail in two directions. Too permissive — no dedup or weak dedup — creates duplicate side effects. Too aggressive — over-zealous dedup — silently swallows legitimate operations. Both are costly.

### The cost of too-permissive deduplication (or none at all)

**Financial cost.** Orphaned records from saga re-initiation triggered double billing notifications for over 200 customers in one reported incident.[^2] Survey data shows 92% of organizations implementing agentic AI reported costs higher than expected, with runaway retry loops as a primary driver.[^5] A single generate-validate-fail loop consumed $180 in tokens over twenty minutes before a human noticed.[^6] These are single-agent costs; multiply by organizational scale for multi-agent systems.

**Trust cost.** A customer who receives a double charge loses trust — not in the agent, but in the organization. The remediation cost — refund processing, customer communication, compliance review — exceeds the original transaction cost by an order of magnitude. Trust erosion compounds: a customer who experiences a duplicate charge scrutinizes every subsequent interaction.

**Operational cost.** Duplicate side effects are typically discovered late. The investigation time to identify, quantify, and remediate duplicates across systems of record is substantial. Orphaned records propagate: a duplicate customer record creates duplicate downstream records — invoices, contacts, audit entries — that each require separate remediation.

### The cost of too-aggressive deduplication

**Silent operation loss.** Content-hash false positives silently swallow legitimate operations. A customer who intentionally places two separate orders for the same item — one as a gift, one for themselves — has the second order deduplicated because the parameters are identical: same product, same quantity, same account. A trader who submits two legitimate buy orders for the same instrument at the same price within a short window has the second order rejected as a duplicate. The failure mode is silence — no error, no retry, just a missing operation that surfaces only when someone notices the downstream absence. This is the mirror image of the duplicate: instead of an unwanted extra operation, a wanted operation disappears.

**Window-sizing failures.** A deduplication window that is too long blocks legitimate re-operations. A customer who cancels and re-books the same room within the window has the re-booking deduplicated. A patient whose dosage is intentionally adjusted within the window has the new prescription treated as a duplicate of the old one. The inverse of the "too short" problem — duplicates escape the window — is the "too long" problem: legitimate operations are captured by it. There is no universally correct window. The right duration depends on the operation's domain semantics and must be calibrated per mutation type.

Idempotency is not a resilience optimization. It is the architectural commitment that makes probabilistic callers safe to deploy against systems of record — and like every architectural commitment, it must be calibrated, not just applied.

---

## Lineage

Factor V draws from four traditions.

The Two Generals Problem proved that no finite protocol can guarantee mutual confirmation over an unreliable channel.[^7] Factor V takes a specific orientation to this impossibility: because a sender can never be certain a message was processed, it must retry, and therefore the receiver must make retries safe. Pat Helland established that as systems scale beyond single-database transactions, idempotency becomes mandatory[^8] — that duplicate elimination belongs in infrastructure rather than in application code[^9] — and that your communication partner may experience "amnesia" as a result of failures or poorly managed state.[^10] Factor V makes Helland's infrastructure imperative non-negotiable, because a probabilistic caller doesn't occasionally experience amnesia — amnesia is its permanent condition.

The caller-side idempotency pattern that most teams know today comes from the API idempotency tradition. Stripe's idempotency key design is the canonical formulation of client-generated tokens for APIs.[^11] The IETF formalized this as the Idempotency-Key header.[^12] Both encode four caller-side responsibilities: generate a unique key per logical operation, persist it across retries, send identical parameters with the same key, and know that a request is a retry. A probabilistic caller satisfies none of these — the assumptions fail simultaneously, not incrementally.

The durable execution tradition shows idempotency ownership moving progressively from caller to infrastructure: from requiring the caller to supply meaningful workflow IDs[^14], to journal-based replay where the runtime generates side-effect tokens[^15], to declarative key derivation where the developer declares what constitutes uniqueness and the framework enforces how.[^16] Factor V completes the arc — when the caller is probabilistic, none of the ownership can remain with the caller.

At least four sources between 2024 and 2026 independently converged on the same diagnosis: a "Decision Intelligence Runtime" where the runtime derives idempotency keys[^3], multi-tiered idempotency that protects the system when the caller is unreliable[^17], an "Action Plane" that enforces idempotency at the infrastructure level[^18], and a mathematical argument that idempotency is fundamentally at odds with stochastic systems.[^19]

[^1]: Cemri, M. et al. (2025). "Multi-Agent System Testing." UC Berkeley. 1,642 execution traces across 7 frameworks. NeurIPS 2025.

[^2]: Practitioner report. Saga re-initiation producing orphaned records: "over 200 orphaned records a week after deployment… each one a customer who had received double billing notifications."

[^3]: Sundaramoorthy, S. (2025). "Idempotency in AI Agents: Why Your LLM Needs a Memory It Can't Hallucinate." March 27, 2025. https://pub.towardsai.net/idempotency-in-ai-agents-why-your-llm-needs-a-memory-it-cant-hallucinate-73e0b4108b75

[^4]: Verma, N. (2025). "LLMs as Unreliable Narrators: Dealing with UUID Hallucination." November 14, 2025. https://nikhil-verma.com/blog/llms-unreliable-narrators-uuid-hallucination/

[^5]: IDC/DataRobot survey. 92% of organizations implementing agentic AI reported costs higher than expected.

[^6]: Kevin Tan. Practitioner report on generate-validate-fail loop. "$180 in tokens… twenty minutes… it never occurred to the agent to stop."

[^7]: Gray, J. (1978). "Notes on Data Base Operating Systems." *Lecture Notes in Computer Science*, vol. 60, Springer, pp. 393–481. Problem first described by Akkoyunlu, Ekanadham, and Huber (SOSP 1975).

[^8]: Helland, P. (2007/2016). "Life beyond Distributed Transactions: an Apostate's Opinion." CIDR 2007; revised for *ACM Queue*, Vol. 14, No. 5, 2016. https://queue.acm.org/detail.cfm?id=3025012

[^9]: Helland (2007), §3.3.

[^10]: Helland, P. (2012). "Idempotence Is Not a Medical Condition." *Communications of the ACM*, Vol. 55, No. 5, May 2012, pp. 56–65.

[^11]: Leach, B. (2017). "Designing robust and predictable APIs with idempotency." Stripe Engineering Blog, February 22, 2017. https://stripe.com/blog/idempotency

[^12]: Jena, J. & Dalal, S. (2025). "The Idempotency-Key HTTP Header Field." Internet-Draft draft-ietf-httpapi-idempotency-key-header-07, October 15, 2025. https://datatracker.ietf.org/doc/html/draft-ietf-httpapi-idempotency-key-header-07

[^14]: Tenzer, K. & Smith, J. (2024). "Understanding Idempotency in Distributed Systems." Temporal Blog, February 27, 2024. https://temporal.io/blog/idempotency-and-durable-execution

[^15]: Restate Documentation. "Key Concepts: Durable Execution." https://docs.restate.dev/concepts/durable_execution/. Vanlightly, J. (2025). "Demystifying Determinism in Durable Execution." November 24, 2025. https://jack-vanlightly.com/blog/2025/11/24/demystifying-determinism-in-durable-execution

[^16]: Inngest Documentation. "Handling Idempotency." https://www.inngest.com/docs/guides/handling-idempotency

[^17]: Srinivasan, B. (2025). "Reliable AI Starts with Idempotency, Not Bigger Models." https://balaaagi.in/posts/reliable-ai-starts-with-idempotency-not-bigger-models/

[^18]: Composio. (2025). "Outgrowing Zapier, Make, and n8n for AI Agents." https://composio.dev/content/outgrowing-make-zapier-n8n-ai-agents

[^19]: Preetham, F. (2024–2025). "Can AI Agents Exhibit Idempotency?" *Autonomous Agents* (Medium). https://medium.com/autonomous-agents/can-ai-agents-exhibit-idempotency-2cea33cc681c

---

## Related Factors

For how intent-aligned interfaces prevent multiple tool paths to the same irreversible mutation, see **Factor III: Intent-Based Communication**. For how access scoping prevents multiple callers from independently initiating the same operation, see **Factor IV: Bounded Access**. For how mutations enter the control plane, see **Factor II: Deterministic Mutations**. For what the control plane returns when operations fail — including the retryability signal that Factor V's stop enforcement consumes — see **Factor VI: Recovery Contracts**. For how the control plane records every action including deduplicated retries, see **Factor VII: Structural Observability**. For how governance scales with consequence, see **Factor I: Governed Operations**.
