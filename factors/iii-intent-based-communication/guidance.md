# Factor III: Intent-Based Communication — Extended Guidance

## Desired Outcome

When this factor is fully implemented, the reasoning layer accomplishes work through a focused set of operations named for what they accomplish — easily distinguishable, invocable with identifiers available from runtime context, returning domain-level outcomes the reasoning layer can act on and communicate. The model selects the right tool more often because fewer tools with clearer names reduce the opportunity for misselection. Implementation details cannot be exfiltrated from the caller's namespace because no operation reveals which vendor, API, or backend system fulfills the operation — not in its name, not in its parameters, and not in its response. More complex workflows can be entrusted to the reasoning layer because orchestration complexity is invisible to the caller — the control plane handles it. The reasoning layer's context budget goes to reasoning, not to mediating concerns the control plane should own.

---

## Terms of Art

The industry has no shared vocabulary for the distinction between a tool that expresses the caller's intent and one that exposes implementation details. The MCP specification calls both "tools." The A2A specification calls one pattern "skills." Neither protocol names the failure mode. Without precise language, teams cannot diagnose their own namespaces, code reviews cannot flag the problem by name, and architectural standards cannot require the healthy pattern.

### Intent-aligned

**Definition.** An implementation that expresses what the operation accomplishes in the domain's language, accepts only contextual identifiers, and hides which systems fulfill it. The implementation survives a backend substitution without changing.

An implementation is intent-aligned when it meets all of the following criteria:

1. The name expresses what the operation accomplishes, not how it is implemented. (`check_out_guest`, not `stripe_create_charge`.)
2. The parameters accept contextual identifiers — values appropriate to the caller's role and available from the operational situation, such as an email address, a booking reference, a service name, or a date range — not system-specific identifiers from backend systems.
3. The implementation hides which systems are involved, what APIs are called, and how operations are sequenced internally. Adding or replacing a backend system does not change what the caller sees.
4. The response returns domain-level outcomes the caller can act on and communicate — not system-specific objects, backend identifiers, or raw API responses from the systems that fulfilled the operation.

**What gap this fills.** The industry has no term for the design target when building implementations for probabilistic callers. "Good API design" is directionally correct but misses the specific constraint: the caller will reason about every detail the implementation exposes, act on information it was never designed to interpret, and exfiltrate implementation details through its outputs. The design surface is new because the caller type is new. "Intent-aligned" names the positive pattern — the thing to build toward — in a single term a team can use in a code review, an architecture decision record, or a namespace audit.

**Diagnostic test.** The four criteria above are the test. For each operation in the namespace: does the name express what the operation accomplishes? Do the parameters use contextual identifiers? Does the implementation hide what fulfills it? Does the response return domain-level outcomes? An operation that passes all four is intent-aligned. An operation that fails any is not.

**Why this matters.** Without a shared term for the design target, teams describe intent-aligned implementations through negation — "don't expose raw APIs," "don't leak vendor names" — but have no positive vocabulary for what healthy looks like. Code reviews can flag violations but cannot name the standard. Architecture documents describe the desired pattern in a paragraph each time rather than referencing a term. The industry needs a word for the thing to build, not just the thing to avoid.

### Implementation-leaked

**Definition.** An implementation that exposes details about the systems that fulfill operations. The implementation often requires refactoring when systems change. This implementation becomes an exfiltration risk when exposed to a probabilistic caller.

Implementation leakage is present when both of the following are true:

1. The implementation reveals details about the systems that fulfill the operation — a vendor or backend system in its name, system-specific identifiers in its parameters, a backend schema in its data model, internal sequencing in its description, or system-specific objects in its response.
2. It is visible to a probabilistic caller. Implementation details that were a maintainability concern between deterministic services become two distinct problems when a probabilistic caller can see them: noise that consumes context and reasoning capacity on work that is not the caller's job, and exposure surface that a compromisable-by-nature caller can be induced to exfiltrate or act on.

**What gap this fills.** The industry has no term for the specific condition of exposing backend implementation details to a caller that will reason about them, act on them, and exfiltrate them. "Bad API design" is too vague — it covers everything from poor naming to missing documentation. "Information hiding violation" is closer but misses the probabilistic caller dimension. An information hiding violation between two deterministic services is a maintainability problem. The same violation exposed to a probabilistic caller is a reliability, security, and capability problem simultaneously. The phenomenon is new because the caller type is new.

**Diagnostic test.** The two criteria above are the test, applied as a conjunction. First: does the implementation reveal implementation details (vendor names, system identifiers, backend schemas, internal sequencing) through its name, parameters, or description? Second: is it visible to a probabilistic caller? Both must be true. An implementation-aligned design behind the boundary — visible only to deterministic code in the control plane — is not necessarily implementation-leaked. The same design exposed to a reasoning layer is.

**Why this matters.** Without this term, teams cannot distinguish between two fundamentally different architectural positions: an implementation-aligned design used appropriately behind the boundary (healthy — this is where system-specific details belong) and the same design exposed to a probabilistic caller (unhealthy — this is the violation). The term makes the violation visible and nameable. A team lead can say "this tool is implementation-leaked" in a code review and point at a specific, testable condition — not a vague design preference.

---

## How This Works

Factor III does not argue that every interface must eliminate implementation details — only that implementation leakage to the reasoning layer be eliminated through control plane design. All implementations exposed to the reasoning layer should be intent-aligned. Four mechanisms make this work in practice.

**Identifier resolution.** The caller provides a contextual identifier — an email address from a guest interaction, a booking reference from a task delegation, a service name from an alert, a deployment ID from a release pipeline. The control plane maps that identifier to whatever system-specific values the backend requires: a Salesforce Contact ID, a Stripe Customer ID, a PMS reservation code, a warehouse zone identifier. This mapping is deterministic code — a lookup, a query, a registry call. The caller never sees the system identifiers and cannot construct them. When a backend system changes its identifier scheme (Salesforce migrates from 15-character to 18-character IDs, a PMS vendor changes its reservation code format), the resolution logic updates behind the boundary. The caller's interface is unchanged.

**Composition.** A single intent-aligned tool often composes multiple backend operations. `check_out_guest` might involve a payment charge, a CRM record update, a room status change, a folio generation, and a confirmation notification — five operations across three or four vendor systems. The control plane handles the sequencing, passes system identifiers between steps (the Stripe charge ID feeds into the folio record; the PMS room status change triggers the housekeeping queue), and manages partial failures. If the payment succeeds but the CRM update fails, the control plane owns the compensation logic — retrying, rolling back, or flagging for manual intervention. The caller receives a structured result describing the outcome (Factor VI), not a raw error from one of several backend systems.

**Description design.** The tool description is the primary mechanism through which a probabilistic caller decides whether and how to invoke a tool. A probabilistic caller is optimized for helpfulness over accuracy — its default response to ambiguity is to try harder, not to stop. A well-designed description works *with* this behavioral tendency by removing ambiguity: it names the outcome ("Checks out a guest: processes final payment, updates the reservation record, and sends a confirmation to the guest's email"), states what the tool expects and returns in domain terms, and describes when to use it — not the technical preconditions. It does not reference which backend systems fulfill the operation, what APIs are called, or what internal sequencing occurs. A description that leaks implementation details gives a probabilistic caller material to reason about that it was never designed to interpret — and it will reason about it, because that is what probabilistic callers do. A description that runs longer than the tool's actual output is a signal that the interface is carrying complexity that belongs behind the boundary.

**Response design.** The return path mirrors the request path: the control plane translates system-specific results into domain-level outcomes before they reach the reasoning layer. A `check_out_guest` that returns `{ stripe_charge_id: "ch_1a2b3c", salesforce_contact_id: "003xxx", pms_status_code: 4 }` leaks implementation through the response even if the name, parameters, and description are intent-aligned. The intent-aligned response: confirmation that the guest is checked out, the total charged, the folio reference, the confirmation email status — domain outcomes the reasoning layer can communicate to the user or use to decide what to do next. The same principle applies to `order_medication` (confirmation, estimated fill time, prior authorization status — not Epic order IDs and FDB interaction codes) and to `reroute_shipment` (new estimated delivery, carrier confirmation — not FedEx tracking API responses and SAP inventory transaction codes). The control plane owns the translation in both directions: contextual identifiers in, domain-level outcomes out. What the reasoning layer never sees on the way in, it should never see on the way out.

These four mechanisms — identifier resolution, composition, description design, and response design — are where the principle meets implementation. The diagnostic framework that follows formalizes the evaluation: do inputs use contextual identifiers resolved behind the boundary? Do outputs return domain-level outcomes rather than system-specific objects? Do descriptions reinforce intent rather than leaking implementation? Is multi-step composition handled by the control plane?

---

## When This Applies (and When It Doesn't)

Factor III applies broadly — any system where a probabilistic caller evaluates and invokes tools benefits from intent-aligned designs. But two contexts change how the guidance applies.

**Deterministic-only callers.** If your system has no probabilistic caller, the principle is sound but this document is not where you will find the most actionable guidance. A cron job, a workflow engine with hard-coded tool selection, a CI/CD pipeline — none of these evaluate namespaces as decisions, consume descriptions as context, or reason about parameters in a context window that can be prompt-injected. The underlying interface design principles still hold (Parnas's information hiding, Cockburn's port naming), but those principles have fifty years of software architecture literature behind them. The diagnostic framework, anti-patterns, and tradeoffs in this guidance are calibrated specifically for probabilistic callers.

**Prototyping and exploration.** When a team is still discovering what operations the system needs to support — before the namespace is stable — investing in intent-aligned boundaries can be premature optimization. Build the implementation-aligned version first, learn what the actual operations are by observing what the model tries to do. Then refactor to intent-aligned boundaries once the operations are understood. The principle describes where to end up, not where to start.

In practice, getting composition boundaries right is genuinely difficult — even with deep domain expertise. The right boundaries between intent-aligned tools and the implementation they hide are discovered through iteration with a probabilistic caller, real users, and production-like conditions. Teams consistently find that their first pass has not abstracted enough complexity out of the reasoning layer: the model still struggles with parameters it should not see, or users trigger workflows the original composition did not anticipate, or a failure mode that seemed rare turns out to be common. This is normal. The architecture should expect to refine composition boundaries repeatedly as the system matures — each iteration moving implementation detail that creates noise or exposure out of the reasoning layer, and ensuring that the contextual information the reasoning layer *does* see is sufficient for it to select the right operation, provide the right identifiers, and handle the outcomes. The goal is not to starve the reasoning layer of information. The goal is to give the reasoning layer exactly what it needs to do its job — the more complex operations the Desired Outcome promises — and leak nothing that belongs to the control plane.

---

## Diagnostic Framework

Evaluating a namespace against this factor requires checking five dimensions: naming, inputs, outputs, descriptions, and composition. Most production tools fail on some dimensions and pass on others. The dimensions are ordered to follow the natural evaluation sequence when examining a tool.

### Five dimensions of evaluation

**Dimension 1: Naming** *(fastest signal, shallowest fix).* Does the name express what the operation accomplishes, or does it reveal how the operation is implemented?

Some implementation-aligned names can appear to pass this test because they describe an operational outcome: `stripe_create_charge` does describe what happens — a charge is created. However, the name fails because it reveals a specific vendor and, almost certainly, the API call behind it. Two tests expose naming failures:

The first is **backend substitution**: could the name survive a backend change? If the payment processor switches from Stripe to Adyen, `stripe_create_charge` breaks. `process_payment` does not.

The second is **domain language**: does the name match how a domain practitioner describes the work? A hotel front desk agent says "check out the guest," not "create a charge." A nurse says "order the medication," not "create a prescription record." An SRE says "rotate the credentials," not "call the secrets API." Names that survive backend substitution but still describe implementation mechanics in vendor-neutral terms — `create_charge`, `insert_record`, `execute_query` — fail this test. The name should reflect the operation as the domain describes it, not the technical action.

**Dimension 2: Inputs** *(where renamed API bindings reveal themselves).* Does the implementation accept contextual identifiers — values the caller naturally possesses from the operational situation — or does it require system-specific identifiers from backend systems?

A tool that accepts an email address, a booking reference, or a date range passes on inputs. A tool that accepts a `customer_stripe_id` or a `stripe_payment_method_id` fails — the caller must obtain values that exist only inside a backend system. The input dimension is where renamed API bindings reveal themselves: `process_payment` sounds healthy until the parameter list requires a Stripe Customer ID and a Stripe Payment Method ID, exposing the payment processor underneath.

**Dimension 3: Outputs** *(most frequently missed — teams that get the request path right often neglect the return path).* Does the response return domain-level outcomes, or does it expose system-specific objects from the backends that fulfilled the operation?

A tool that returns "guest checked out, $247.50 charged, confirmation sent to guest@example.com" passes on outputs. A tool that returns a Stripe charge object, a Salesforce API response, and a PMS status code fails — the reasoning layer receives implementation details it must interpret, can exfiltrate, and will attempt to surface to the user. The output dimension is where intent-aligned implementations most commonly break down, because teams focus on the request path (what the caller sends) and treat the response as "whatever the backend returns." The control plane's job is translation in both directions: contextual identifiers in, domain-level outcomes out.

**Dimension 4: Descriptions** *(most commonly overlooked on the request path).* Does the tool description tell the caller what the tool accomplishes and when to use it — or does it leak implementation details the name and inputs concealed?

A tool's description (or the equivalent in A2A: the skill description, examples, and tags) is the primary mechanism through which a probabilistic caller decides whether and how to invoke it. A tool can pass on naming, inputs, and composition and still leak implementation through its description — or have a description so vague the caller cannot evaluate it at all.

Descriptions leak implementation when they reference vendor names or backend systems the interface otherwise hides ("Calls the Stripe Charges API to process payment"), include sequencing instructions that belong behind the boundary ("Call this after creating the Stripe customer"), expose system-specific concepts the inputs do not require ("Uses the PMS folio subsystem to generate a guest invoice"), or document internal error codes the caller cannot act on ("Returns PMS error code 4012 if the room is in maintenance state").

Descriptions fail in the other direction when they are too vague to be actionable. "Processes a checkout" tells the caller nothing about what inputs are needed, what the operation accomplishes, or what outcomes to expect. A probabilistic caller — optimized for helpfulness over accuracy — evaluating a vague description will either avoid the tool (underuse) or invoke it speculatively (misuse), and its tendency to try harder rather than stop means speculative invocation is the more likely outcome.

In A2A, skill descriptions, examples, and tags serve the same function. A skill described as "Handles hotel checkout using Stripe, Salesforce, and PMS" leaks three vendor names. The same skill described as "Checks out a guest given a booking reference — handles payment, guest records, and confirmation" is intent-aligned. The `examples` field in an AgentSkill provides the same opportunity and the same risk: examples that reference system identifiers or vendor-specific workflows leak implementation; examples that use domain language and contextual identifiers reinforce intent alignment.

**Dimension 5: Composition** *(hardest to get right).* Does completing a single operation require the caller to invoke multiple tools in a specific sequence?

A single `check_out_guest` tool that composes payment, CRM update, room status change, folio generation, and notification internally passes on composition. Five separate tools — `create_stripe_charge`, `update_salesforce_contact`, `update_pms_room_status`, `generate_folio`, `send_confirmation_email` — for the same operation fail. The violation is not in any individual tool. It is in the namespace as a whole: the caller has been made the integration engineer, responsible for sequencing, passing identifiers between steps, and handling partial failures across service boundaries.

---

## Named Anti-Patterns

Five failure modes appear repeatedly in production namespaces, one for each diagnostic dimension. Each has a name, a diagnostic test, and a structural fix. They are ordered to match the diagnostic dimensions.

**1. The raw API as tool.** A vendor API endpoint is exposed as an agent-callable tool without abstraction. `stripe_create_charge`, `salesforce_soql_search`, `update_pms_room_status` — the tool name reveals which vendor, which system, and almost certainly which API operation. The caller sees the implementation before evaluating anything else. This is the most recognizable Factor III violation and the clearest naming failure.

*Diagnostic:* Does the tool name contain a vendor name, a backend system, or a verb that describes an API operation rather than what the operation accomplishes?

*Fix:* The API call becomes an implementation-aligned operation behind the intent boundary. The tool the caller sees is named for the outcome it accomplishes.

**2. The wrapped API as tool.** The tool has been renamed to sound like a capability, but the inputs still require system-specific identifiers. `process_payment` sounds healthy — until the parameter list requires `customer_stripe_id` and `stripe_payment_method_id`. The name improved; the boundary did not move. This is the most deceptive pattern and the signature input failure, because the intent-aligned name suggests the implementation has been redesigned when only the label changed.

*Diagnostic:* Does the tool accept identifiers that only exist inside a backend system, despite having an intent-aligned name?

*Fix:* The control plane resolves contextual identifiers to system identifiers behind the boundary. The caller provides an email address or booking reference; the control plane maps it to whatever system-specific identifiers the backend requires.

**3. The leaky response.** The tool passes on naming and inputs — but the response hands back raw backend objects. `check_out_guest` returns a Stripe charge object, a Salesforce update timestamp, and a PMS status code. The reasoning layer receives system-specific details the request path successfully hid, and will attempt to interpret them, surface them to the user, or store them in context where they become exfiltration targets. This is the most frequently missed failure because teams that invest in intent-aligned request design often treat the response as "whatever the backend returns, passed through."

*Diagnostic:* Does the response contain system-specific identifiers, backend object schemas, vendor-specific status codes, or raw API responses from the systems that fulfilled the operation?

*Fix:* The control plane translates backend results into domain-level outcomes before they reach the reasoning layer. The response should contain what the caller needs to communicate results or decide what to do next — in the domain's language, at the domain's level of abstraction.

**4. The leaky description.** The tool passes on naming and inputs — but the description references vendor names, backend systems, or implementation details the interface otherwise conceals. This is especially common alongside the wrapped API pattern: teams rename the tool and update the parameters but leave the description unchanged, reintroducing the vendor details the new name was meant to hide. A description that reads "Calls the Stripe Charges API to process payment for the given customer" undoes the work of naming the tool `process_payment`. A probabilistic caller will read the description and reason about Stripe — regardless of what the name and inputs suggest. This is the most commonly overlooked failure on the request path.

*Diagnostic:* Does the description reference systems, vendors, sequencing, or implementation concepts that the name and inputs do not expose? Is the description so vague that the caller cannot evaluate whether or when to invoke the tool?

*Fix:* Rewrite the description in domain language: what the tool accomplishes, what it expects, what it returns, and when to use it — without referencing which systems fulfill the operation.

**5. The flat namespace / agents as orchestrators.** Five separate tools — `create_stripe_charge`, `update_salesforce_contact`, `update_pms_room_status`, `generate_folio`, `send_confirmation_email` — for what should be a single `check_out_guest`. The violation is not in any individual tool. It is in the namespace as a whole: the caller has been made the integration engineer, responsible for sequencing, passing identifiers between steps, and handling partial failures across service boundaries. The same failure occurs at the agent level when a team treats a remote agent as a single function call — invoking it with implementation-level parameters and expecting a return value — instead of delegating to it as a capability that manages its own work. Whether the composition failure is among tools or among agents, the result is the same: the caller is assembling implementation-level operations that should be composed behind the boundary. This is the hardest failure to fix because composition does not exist yet — it must be built into the control plane.

---

## Industry Examples

Factor III's hotel checkout exemplar demonstrates the principle in hospitality. The same architectural pattern — and the same failure modes — appear across industries wherever an agent must coordinate multiple backend systems to accomplish an operation.

### Healthcare — Medication Order Entry

An implementation-leaked namespace for computerized physician order entry might expose: `query_epic_patient_record`, `check_drug_interaction_fdb`, `verify_formulary_coverage_surescripts`, `create_prescription_epic`, `submit_prior_auth_covermymeds`, and `send_prescription_surescripts`. Six tools across four vendor systems (Epic, First Databank, SureScripts, CoverMyMeds), each requiring system-specific identifiers the caller has no business handling.

The intent-aligned alternative: `order_medication`, accepting a patient identifier, medication name, and dosage. The control plane composes the drug interaction check, formulary verification, prescription creation, and routing to pharmacy or prior authorization as the clinical workflow requires. The caller expresses clinical intent; the control plane handles vendor orchestration. The response returns what the reasoning layer needs to communicate to the clinician: order confirmation, estimated fill time, whether prior authorization is required — not Epic order IDs, FDB interaction codes, or SureScripts transaction references.

Patient safety makes this the sharpest illustration of the principle. Drug interaction checks are non-negotiable sequencing — a reasoning layer that skips or misorients this step creates clinical risk. HIPAA requires that system identifiers like Epic medical record numbers and FDB drug codes remain behind the boundary. The single intent-aligned implementation hides six vendor-specific operations behind a boundary that serves usability, safety, and regulatory compliance simultaneously.

### Logistics — Shipment Rerouting

An implementation-leaked namespace: `get_shipment_status_fedex`, `cancel_fedex_pickup`, `create_ups_shipment`, `update_inventory_sap`, `notify_warehouse_wms`, `update_customer_order_shopify`, `recalculate_delivery_eta`. Seven tools across five systems (FedEx, UPS, SAP, a warehouse management system, Shopify), requiring the caller to know which carrier currently holds the shipment, how to cancel one carrier's pickup before booking another, and how to propagate changes across inventory, warehouse, and customer-facing systems.

The intent-aligned alternative: `reroute_shipment`, accepting an order reference and a new destination or carrier preference. The control plane determines whether the current carrier requires cancellation, books the new shipment, adjusts inventory records, notifies the warehouse, updates the customer order, and recalculates the delivery estimate. The response: new estimated delivery date, carrier confirmation, and customer notification status — not FedEx cancellation receipts, UPS booking IDs, or SAP inventory transaction codes.

The rerouting workflow has conditional branches that deterministic code handles reliably: does the current carrier need active cancellation, or has the shipment not yet been picked up? Is the new destination served by the same warehouse zone, or does inventory need to transfer? A probabilistic caller sequencing seven tools would need to evaluate each branch correctly. The control plane handles these branches as deterministic logic behind the intent boundary, and the caller never sees the decision tree.

---

## Tradeoffs and Tensions

Intent-aligned boundaries are not free. Five costs are worth naming honestly, ordered from the most straightforward to address to the most architecturally challenging.

**Naming requires domain expertise.** Finding the right intent-aligned name requires understanding the domain — not just the backend systems. Teams without access to domain practitioners, or without an established ubiquitous language, may produce names that are neither implementation-aligned nor truly intent-aligned: vague abstractions like `process_action` or `handle_request` that pass the naming test (no vendor name, no API verb) but fail the domain language test (a domain practitioner would not use these words to describe the operation). The fix is not to abandon intent-aligned naming but to invest in domain understanding first — talk to the people who do the work, learn what they call the operations, and use their language. This is Evans's ubiquitous language argument applied to tool design.

**Coupling shifts, not disappears.** When operations change — checkout now requires loyalty point redemption, medication ordering now requires prior authorization — the intent-aligned interface may not change, but the control plane's composition logic must. The coupling between process and implementation has moved from the caller (where it was visible and fragile) to the control plane (where it is governed and testable). This is a better place for the coupling to live. But it is still coupling, and teams that assume intent-aligned boundaries eliminate coupling rather than relocating it will be surprised when process changes require control plane updates.

**Abstraction has a cost.** Building the translation layer — identifier resolution, multi-step composition, compensation logic, structured result contracts — is real engineering work. So is reorganizing how teams discover and reuse tools: most service registries and catalogs are organized by backend system (Stripe services, Salesforce services, PMS services), but intent-aligned boundaries require organizing by domain (guest operations, payment operations, inventory operations, compute operations). Teams cannot rely on engineering-oriented service discovery patterns for this layer — the taxonomy has to match the domain, not the infrastructure. For simple systems with a single backend and a single caller, the abstraction may cost more than it saves in the near term. The guidance on prototyping contexts applies here: the translation layer is an investment that pays off as backends multiply, callers diversify, and the namespace grows.

**Observability improves, not degrades.** A common objection: highly composed intent-aligned tools produce less granular caller-side telemetry. When `check_out_guest` fails, the caller knows it failed but not which backend step failed. But this framing confuses caller-side visibility with system-wide visibility. Moving composition into the control plane means a dedicated architectural layer now mediates every backend operation — and that layer can emit structured telemetry that traces the full operation through every step, with a coherence no scattered agent-side logging could match. The caller receives domain-level outcomes (Factor VI addresses what the caller needs to retry or recover). Operators see implementation-level traces. Safe Retries (Factor V) addresses what the probabilistic caller needs to handle failure. Structural Observability (Factor VII) addresses why the control plane architecture produces *better* visibility than the alternative — not worse. These are different audiences seeing different views of the same operation, and the architecture serves both better than the implementation-leaked alternative where observability depends on the reasoning layer reporting what it did.

**Over-composition is its own failure mode.** The instinct to hide all complexity behind the boundary produces two pathologies. The first is monolithic orchestration: a single tool that attempts to handle every possible state change, every edge case, every failure path — massive saga patterns composed into an omniscient workflow. The second is the traditional microservices weakness applied to the control plane: so many layers of decomposition behind the intent-aligned interface that the system becomes byzantine and impossible to maintain. The intent boundary is clean, but getting from `check_out_guest` to the actual backend calls traverses six abstraction layers that no single engineer can hold in their head. Both are composition failures — one composes too much into a single boundary, the other decomposes too much behind it. Neither is designed for comprehension and maintainability.

The principle is that the caller never handles *implementation-level* sequencing — but it can absolutely handle *operation-level* recovery decisions between intent-aligned tools. For how the control plane communicates failure back to the caller in structured, actionable terms, see Recovery Contracts (Factor VI). For how the caller safely retries operations without creating duplicate side effects, see Safe Retries (Factor V). For how the caller's namespace is scoped to only the operations its role requires, see Bounded Access (Factor IV).

---

## The Cost of Implementation Leakage

Implementation-leaked namespaces impose four distinct costs. Each is independently measurable; together they compound.

**Token economics.** Every operation in the namespace consumes context window tokens — its name, description, parameter schema, and type definitions. An implementation-leaked namespace multiplies this cost: five operations where one would suffice, each carrying vendor-specific parameter schemas with enumerated values and sequencing constraints. But the static cost is the smaller problem. When the reasoning layer is forced to guess at orchestration — which operation to call next, what identifiers to pass between steps, how to handle a partial failure mid-sequence — it burns context on reasoning it was never designed to do well, and the sessions that fail trigger retries that burn it again. Implementation-leaked namespaces do not just consume more context at rest; they consume more context in motion, and the failures consume it again.

**User experience degradation.** An agent operating through implementation-leaked tools produces experiences that feel mechanical, high-friction, and unhelpful — the opposite of what users expect from an AI-powered system. The reasoning layer cannot compose a fluid, coherent interaction when it is busy chaining raw API calls. The difference between `stripe_create_charge` → `update_salesforce_contact` → `update_pms_room_status` and a single `check_out_guest` is not just architectural — it is the difference between a user experience that feels like talking to a database and one that feels like talking to a concierge.

**Security risk.** Every implementation detail visible to the reasoning layer is a detail that can be exfiltrated, manipulated, or exploited. Vendor names, system identifiers, backend schemas, and internal sequencing exposed through tool interfaces become attack surface. For how bounded access controls the blast radius of what tools exist in the caller's namespace, see Factor IV: Bounded Access.

**Underperforming agents.** With implementation-leaked tools, agents stay limited — not because the model lacks capability, but because the namespace constrains what the model can accomplish. An agent that must orchestrate five vendor-specific API calls to check out a guest cannot reliably handle a more complex request like "check out the guest, apply their loyalty points, and rebook them for next month." An agent operating through intent-aligned tools can, because the orchestration complexity is handled by the control plane and the model's context is available for the actual work. Implementation-leaked namespaces cap agent capability at the level of an integration engineer. Intent-aligned namespaces let the agent operate at the level of the domain.

---

## Lineage

The principle that interfaces should express intent rather than implementation has deep roots in software architecture. Parnas's information-hiding criterion (1972) established that module boundaries should hide decisions likely to change.[^1] Cockburn's Hexagonal Architecture (2005) applied this to system boundaries, defining ports as purpose-named interfaces with technology-specific adapters behind them — Cockburn's explicit rule that ports be "named according to what they are for, not according to any technology" is this factor's principle stated at the architectural level.[^2] Martin's clean code tradition (2008, 2017) brought the principle to function and system design: intention-revealing names, single responsibility, the rule that functions should not mix levels of abstraction, and the dependency rule that boundaries protect high-level policy from low-level implementation detail.[^3][^4] Evans's Domain-Driven Design coined the ubiquitous language concept that gives this factor its naming test — `check_out_guest` is ubiquitous language; `stripe_create_charge` is infrastructure vocabulary leaked to the caller.[^5]

Factor III applies the same principle to a new boundary: the interface between a probabilistic reasoning layer and the deterministic systems it operates on. The principle is not new. The nature of the probabilistic caller is.

[^1]: Parnas, D.L. (1972). "On the Criteria To Be Used in Decomposing Systems into Modules." *Communications of the ACM*, 15(12), 1053–1058. https://doi.org/10.1145/361598.361623

[^2]: Cockburn, A. (2005). "Hexagonal Architecture (Ports and Adapters)." https://alistair.cockburn.us/hexagonal-architecture/

[^3]: Martin, R.C. (2008). *Clean Code: A Handbook of Agile Software Craftsmanship.* Prentice Hall.

[^4]: Martin, R.C. (2017). *Clean Architecture: A Craftsman's Guide to Software Structure and Design.* Prentice Hall.

[^5]: Evans, E. (2003). *Domain-Driven Design: Tackling Complexity in the Heart of Software.* Addison-Wesley.

---

## Related Factors

This factor addresses how the caller communicates with the control plane — shaping tool boundaries by intent rather than implementation. For which tools exist in a given caller's namespace, see Factor IV: Bounded Access. For how mutations pass through the control plane once intent is expressed, see Factor II: Deterministic Mutations. For what the control plane returns when an operation fails, see Factor VI: Recovery Contracts.