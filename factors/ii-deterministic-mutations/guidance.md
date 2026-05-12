# Factor II: Deterministic Mutations — Extended Guidance

## Desired Outcome

When this factor is fully implemented, no AI agent directly creates, modifies, or deletes state in a system of record. The reasoning layer expresses intent; the control plane owns everything that happens next. Every mutation passes through deterministic infrastructure that validates preconditions against authoritative state, executes through the appropriate backend API with scoped credentials, and records a structured audit entry — before the operation touches the system of record. Compensation paths exist for every mutation capability, designed and deployed alongside the forward operation. The LLM never sees a connection string, never constructs a query, never handles a raw API response. The organization stops discovering agent-initiated mutations it cannot explain, cannot reverse, and did not authorize.

---

## How This Works

Factor II states: every create, update, and delete against a system of record passes through the control plane. This section makes it operational — what the prohibition actually means, what mediation looks like, and what the control plane provides that justifies the architectural constraint.

### The prohibition

No LLM connects to a system of record with write-authorized principal access — not via a CLI, an API, an MCP server, or a web interface. The LLM does not hold credentials to any system of record. It does not construct queries, assemble payloads, or execute commands against backends. This is not a guideline. It is a hard architectural constraint. The Replit database deletion is what happens when the constraint is absent: the agent had direct access, no boundary existed, and the agent destroyed a production database containing records on 1,206 executives and 1,196 companies — during an active code freeze stated repeatedly in ALL CAPS. The agent then fabricated 4,000 fake records to cover the gap.[^1]

The prohibition follows from a deeper observation about the reasoning layer's competence boundary. The reasoning layer is good at reasoning about intent — understanding context, interpreting what should happen, deciding what to propose. Everything beyond that arena — validation against authoritative state, execution of the mutation, compensation when it fails, audit of what occurred — requires guarantees that a probabilistic system cannot provide. The reasoning layer is optimized for helpfulness over accuracy; its default response to ambiguity is to try harder, not to stop.[^11] That behavioral profile is an asset when reasoning about what a user needs. It is a liability when the next step is a write to a financial ledger. The Replit agent reached the database fine. The prohibition says that reaching the database is not an ability the reasoning layer should be granted, because the guarantees enterprise systems require — deterministic validation, transactional integrity, designed compensation, structured audit — are not things a probabilistic caller can provide.

### The mediation

All mutation traffic flows through the control plane. The control plane is the only component with write-authorized access to enterprise systems of record. The reasoning layer expresses what should happen — using intent-aligned interfaces (Factor III) — and the control plane determines whether it should happen, how it happens, and what to do if it fails.

This mediation is caller-agnostic. A human asking a chatbot to cancel a reservation, an agent delegating a refund to another agent via A2A, a nightly reconciliation job correcting inventory discrepancies — in each case the reasoning layer expresses intent and the control plane owns everything after the boundary. The boundary applies identically regardless of who triggers the mutation.

### What the control plane provides

The mediation layer is not overhead. It is the only place where four things can happen, and each is necessary.

**Deterministic validation.** The control plane checks the reasoning layer's request against the system of record's actual state. The factor paper puts it plainly: "validates that it *should* happen." The reasoning layer's confidence is irrelevant; the system of record is authoritative. Does the reservation the agent wants to cancel actually exist? Is it in a cancellable state? Is the caller authorized to cancel it? These are deterministic checks against authoritative data — not things the reasoning layer can verify from its context window, and not things a well-worded prompt can substitute for.

**Scoped credential management.** The control plane holds and manages credentials to enterprise systems of record. The reasoning layer never sees a connection string, never handles an API key, never touches an OAuth token. Credentials are scoped per-operation, not per-agent — the control plane provisions the minimum access needed for each specific mutation. (Factor IV governs which operations the agent can request; the control plane determines what credentials each operation uses.)

**Compensation orchestration.** Every mutation capability in the control plane ships with a designed compensation path — the reversal or correction that runs when the forward operation fails or is later found to be incorrect. This is a design-time architectural constraint inherited from the Saga tradition (§ Lineage): you don't deploy `charge_credit_card` without also deploying its compensation path. The compensation is a property of the *system*, not a responsibility of the *triggering agent*. A customer-facing chatbot might invoke a charge through the control plane, but `refund_credit_card` might only be available to a staff-level agent, a human supervisor, or a nightly reconciliation job. The control plane routes compensation to an appropriately authorized actor. (Factor IV governs which callers can see which capabilities; Factor II requires that the system as a whole has the compensation path before the forward operation is available.)

**Audit production.** Every mutation that passes through the control plane produces a structured audit record — what was requested, what was validated, what was executed, what the outcome was. This is Factor VII's observability surface, but Factor II's mediation layer is what makes it possible. Without mediation, there is no reliable record of what the agent did or why.

### Multi-step coordination

When a single intent requires mutations across multiple backends — "check out this guest" touching billing, property management, and room status — the control plane orchestrates the sequence with designed compensation at each step. The reasoning layer doesn't coordinate distributed transactions. The control plane does — and each step in the sequence has a compensation path that was deployed alongside the forward operation.

The key architectural commitment: the control plane owns the saga graph. It knows which steps have completed, which have failed, and what compensation is required. The reasoning layer that triggered the checkout does not manage this state and cannot observe intermediate steps. It receives a structured result when the saga completes — success, failure, or partial success with specific information about which steps succeeded and which require attention. (Factor VI addresses how the control plane communicates these outcomes back to the caller.)

### Where the prohibition does not apply

Factor II governs mutations against systems of record — state that is visible to other business processes, subject to regulatory requirements, or irrecoverable without compensation. It does not govern everything the reasoning layer does. Reads are categorically different: hallucinated reads are recoverable; hallucinated writes are not. Agent-internal state — memory updates, scratch space — is the reasoning layer's own workspace and outside the factor's jurisdiction. The scoping question beyond mutations to a system of record — where the boundary sits and why it sits there — is the next section's territory.

---

## When This Applies (and When It Doesn't)

Factor II governs mutations against systems of record. It does not govern everything the reasoning layer does. The boundary matters because the most visible AI agent ecosystem — Devin, Claude Code, Cursor — appears to operate without it, and understanding why requires distinguishing artifact production from runtime mutation.

### What Factor II does not govern

Two categories of reasoning layer output fall outside Factor II's jurisdiction. Recognizing them prevents the factor from overclaiming — and clarifies why the most visible AI agent ecosystems are not counterexamples.

**Artifact production, not runtime mutation.** Writing code, generating configuration files, drafting documents — these produce artifacts, not runtime mutations. A coding agent committing to a feature branch is not mutating production state. It is producing a proposal for future behavior that passes through additional deterministic gates — CI pipelines, code review, merge approval, deployment processes — before it can affect a system of record. What runs in production may not even be the current state of main; deployment pipelines, feature flags, canary releases, and rollback mechanisms all sit between the artifact and the runtime state. Submitting a PR is closer to expressing intent than to executing a mutation. The confinement boundary exists — it is the CI/CD pipeline and deployment process, a control plane native to the software development domain.

The Replit incident illustrates the distinction. The coding agent that deleted a production database was not "writing code" — it had direct runtime access to a production database and executed destructive operations against it. This is precisely an illustration of expanding reasoning layer scope beyond artifact production into runtime mutation — and precisely where Factor II's prohibition applies. A coding agent with direct database access is no longer just writing code — it is mutating systems of record. Staging writes (drafts, sandboxes) belong in the artifact tier too: they produce intermediate artifacts, not consequential mutations.

**Reads and agent-internal state.** Most reads — retrieval, search, information gathering — are outside Factor II's scope. (Factor III addresses how read operations are structured.) Agent-internal state — memory updates, scratch space — is reasoning layer territory. It matters to Factor II only when it becomes part of an intent expression that crosses the mutation boundary; until then, it is the reasoning layer's own workspace and outside this factor's jurisdiction. Text-to-SQL tools (Databricks Genie, Snowflake Intelligence) belong here: the dominant use case is analysts asking natural language questions that translate to `SELECT` queries against data warehouses. The reasoning layer is producing an artifact — a SQL query — that retrieves information. The tension arises in two directions. The first is write access: if these systems are granted `INSERT`, `UPDATE`, `DELETE`, or DDL operations against production databases, the generated SQL is no longer on the read path — it is a runtime mutation against a system of record, and Factor II's prohibition applies. The second is high-risk reads: in regulated contexts, some reads carry governance requirements that make them functionally equivalent to mutations from an audit and risk perspective. A read of HIPAA-regulated patient data must be logged. A read that surfaces PCI-scoped cardholder data may be non-recoverable in the sense that the data, once exposed to the reasoning layer's context, cannot be un-exposed. These reads should route through the control plane — not because they mutate state, but because they share the governance requirement.

Everything else — every create, update, and delete against a system of record, plus the high-risk reads that share the governance requirement — is Factor II's jurisdiction. The rest of this document addresses how.

---

## Diagnostic Framework

Three dimensions of a healthy Factor II implementation, ordered from the most obvious starting point to the subtlest.

**Dimension 1: Primary namespace mutations.** *What can the agent write or delete through its primary namespace — and does every mutation-capable path route through the control plane?* Inventory every tool, skill, CLI, MCP server, and any other capability the agent can invoke or directly access during a transaction. This is the agent's primary namespace. Identify anything in the agent's primary namespace that can create, update, or delete state in a system of record. For each one, confirm it routes through the control plane. This is the most visible surface and the easiest to audit — the team deliberately materialized these capabilities, so the team can trace them. (Factor IV addresses how to bound and govern the primary namespace in depth.)

**Dimension 2: Effective namespace mutations.** *Can the agent accomplish a write or delete you didn't design for within its effective namespace — and does every mutation-capable path route through the control plane?* The agent's effective namespace is everything the agent can actually accomplish at runtime: directly or indirectly. This could be agent-to-agent escalation, tool repurposing, shell or web access, or any other path available to the agent. The gap between the primary namespace and the effective namespace is where mutation surprises can potentially hide. Any mutation that is possible and wasn't designed for belongs in this dimension. (Factor IV addresses effective namespace controls, and how to align effective and primary namespaces as much as possible.)

**Dimension 3: High-risk reads.** *Are there reads in the agent's namespace that are non-recoverable, must be logged, or carry regulatory governance requirements — and have you routed them through the control plane?* Most reads are outside Factor II's scope. But some reads share the governance requirement with mutations: a read of HIPAA-regulated patient data must be logged; a read that surfaces PCI-scoped cardholder data may be non-recoverable in the sense that once the data is exposed to the reasoning layer's context, it cannot be un-exposed. The team that correctly governs every write but treats all reads as categorically safe has not evaluated this surface. The subtlety is that these reads *look* safe — they don't mutate state — but they carry audit, compliance, or exposure consequences that make them functionally equivalent to state changes in their need for governance.

---

## Named Anti-Patterns

Three anti-patterns, each mapping to a diagnostic dimension. Ordered from easiest to spot to most subtle.

**Anti-pattern 1: Too much agency.**

*Diagnostic dimension: Primary namespace mutations.*

The reasoning layer has direct write access to systems of record — not because the team assessed the risk and accepted it, but because the default design assumption was that the agent *should* have broad capabilities. The bias is subtle and pervasive: the ideal end state is imagined as the agent doing more, reaching more, controlling more. Give it database access so it can be flexible. Give it API keys so it can act autonomously. Let it construct queries so it's not constrained by predefined tools. This is a culture and design mentality problem — the wrong processes get migrated or designed around agent capabilities because the team assumes more agency is better.

The Replit database deletion is the canonical incident: the agent had broad access because that was the design, no architectural boundary existed, and the agent destroyed a production database during a code freeze.[^1] The OpenAI Operator purchase is the commercial variant: the agent had direct transactional access and made an unauthorized $31.43 purchase from Instacart when asked only to research egg prices.[^3]

Look for: design conversations that frame agent capability as "how much can the agent do?" rather than "what can the agent reasonably do — and where have we invested the wrong responsibility with the agent vs. the control plane?"; architecture decisions that expand agent scope without a corresponding governance assessment; reasoning layer components that hold write-authorized credentials to systems of record. Fix: the reasoning layer is good at reasoning about intent — let it do that well. The goal is not to box the agent in but to make it responsibly useful: the reasoning layer expresses intent, the control plane owns everything that happens next, and each is doing the work it is competent to do.

**Anti-pattern 2: Shadow paths.**

*Diagnostic dimension: Effective namespace mutations.*

The primary tool inventory routes through the control plane — but mutation capability exists in the effective namespace that the team didn't design for. This goes beyond the V1 concern of "a CLI the agent can shell out to" (though that remains a common variant). The more insidious forms: an agent-to-agent delegation that escalates to a second agent with broader write access than the first agent was granted. A read-oriented tool whose parameters can be repurposed to accomplish a write the team didn't anticipate. An MCP server with broader permissions than the sanctioned tools require, where the agent discovers and invokes capabilities the team never intended to expose.

The common thread: the team audited the primary namespace and it looks correct. The effective namespace — what the agent can actually accomplish — is larger than what the team designed.

Look for: agent-to-agent delegation paths where the downstream agent has write access the upstream agent doesn't; tools whose parameter space allows operations beyond the tool's intended use; MCP servers whose permission scope exceeds the sanctioned tool inventory; any mutation capability the agent can reach that doesn't appear in the team's tool-by-tool audit. Fix: audit the effective namespace, not just the primary namespace. The test is not "do our designed tools route through the control plane?" but "can the agent accomplish a write through any path we haven't evaluated?" (Factor IV addresses effective namespace controls in depth.)

**Anti-pattern 3: The rubber stamp.**

*Diagnostic dimension: High-risk reads.*

The team correctly governs every mutation — writes and deletes route through the control plane, the primary namespace is audited, shadow paths have been traced. But all reads are rubber-stamped as safe without evaluating which ones carry governance requirements. The rubber stamp is the assumption that "read = safe" without actually auditing risk and exposure.

The most insidious variant: reads that surface data into the reasoning layer's context window that should never reach it. A read that retrieves PCI-scoped cardholder data doesn't mutate state — but once that data enters the reasoning layer's context, it cannot be un-exposed. It may be cached, logged, included in downstream tool calls, or surfaced in responses. The read is non-recoverable in the same way a write is: you cannot undo the exposure. Regulated reads (HIPAA patient data, SOX financial records) carry a similar profile — not because the data can't be un-seen, but because the access must be logged, scoped, and auditable. These reads share the governance requirement with mutations and should route through the control plane accordingly.

Look for: reads over regulated or sensitive data that are not routed through the control plane; reads that surface data into the reasoning layer's context without audit logging or minimum-necessary filtering; read operations where the team's risk assessment was "it's a read, not a write" rather than an evaluation of what data is exposed and to whom. Fix: audit the read inventory for governance requirements. Any read that is non-recoverable, must be logged, or carries regulatory consequences should route through the control plane — not because it mutates state, but because it shares the governance requirement. (Factor III addresses how to design intent-aligned interfaces that filter what data reaches the reasoning layer — minimum-necessary responses rather than full record dumps. Factor VI addresses how to structure error responses so that failure paths don't leak sensitive data the success path correctly withheld.)

---

## Industry Examples

Three domains, each illustrating how a specific industry's constraints shape Factor II's application.

### Hospitality: primary vs. effective namespace in a multi-step saga

Hotel guest checkout is a natural multi-step mutation: charge the card, close the folio, update room status, generate the receipt. (Dewy Resort, an open-source sample application used throughout this framework, implements this pattern.) Each step touches a different backend system. Each has a compensation path designed at the same time as the forward operation — reverse the charge, reopen the folio, reset room status.

The domain shapes Factor II's diagnostic dimensions in two ways. First, the primary namespace for the guest-facing chatbot is clear: it can trigger `check_out_guest`. The control plane orchestrates the four underlying mutations in sequence, and if step 3 fails (room status update), compensation runs for steps 1 and 2. The reasoning layer that triggered the checkout does not manage this — it receives a structured result when the saga completes. Second, the effective namespace question surfaces naturally: `reverse_charge` exists in the system but should only be available to the billing supervisor agent and the nightly reconciliation job. The guest-facing chatbot's effective namespace should not include it — but agent-to-agent escalation paths or broadly scoped MCP servers could expose it. (Factor IV governs which callers can see which capabilities.)

### Financial services: the irrecoverability window and high-risk reads

Trade execution is the highest-consequence mutation domain Factor II addresses. The primary namespace includes order submission tools; the control plane validates pre-trade checks — position limits, restricted lists, regulatory holds — deterministically before the order reaches the exchange. No probabilistic system evaluates these constraints. The control plane checks the order against authoritative compliance state, not against the reasoning layer's belief about what the client's position allows.

The compensation path has a time window. Exchange rules may make trade cancellation or correction unavailable after settlement — typically T+1 for equities. The control plane must account for the irrecoverability window: compensation that is available at 10:01 AM may not be available at market close.

The high-risk reads dimension is equally visible here. An agent reading a client's portfolio positions, trade history, or account balances is accessing data subject to regulatory audit requirements. These reads don't mutate state — but they must be logged, scoped to the specific inquiry, and auditable. A read that surfaces a client's full portfolio into the reasoning layer's context when the task only required checking a single position is a Dimension 3 failure: the read was rubber-stamped as safe without evaluating what data was exposed.

### Healthcare: where high-risk reads are the primary surface

Healthcare is the domain where Dimension 3 — high-risk reads — is most visible. An agent reading patient records is accessing HIPAA-regulated data that must be logged, scoped to minimum-necessary access, and auditable. The read doesn't mutate state, but once patient data enters the reasoning layer's context, it cannot be un-exposed — it may be cached, referenced in downstream tool calls, or surfaced in responses to other queries. The control plane must enforce minimum-necessary filtering: the agent sees medications relevant to the current encounter, not the full chart.

The mutation surface matters too — prescription modification requires physician confirmation, not because the architecture demands it but because the domain does. Administrative updates (scheduling, demographics) don't. But the domain's most insidious Factor II risk is not the write path. It is the read path that teams govern less carefully because "it's just a query."

---

## The Cost of Direct Mutation

What happens when the prohibition is absent — when the reasoning layer can reach systems of record directly? Factor II has the richest production incident record of any factor in the framework.

**The Replit database deletion (July 2025).** An AI coding agent deleted an entire production database containing records on 1,206 executives and 1,196 companies — during an active code freeze stated repeatedly in ALL CAPS. The agent then fabricated 4,000 fake records to cover the gap. No architectural boundary existed between reasoning and state. No compensation path. No audit trail of what was destroyed. The agent had the access. The absence of the mutation boundary was the root cause.[^1]

**The OpenAI Operator purchase (February 2025).** Asked only to research egg prices, the agent made an unauthorized $31.43 purchase from Instacart — violating OpenAI's own stated safeguard that Operator would "require user confirmations before completing any significant or irreversible action." System-of-record mutation (financial transaction) executed without control-plane mediation. Textual instructions are not access controls.[^3]

**The Shah et al. fault taxonomy (March 2026).** 385 faults across 40 agentic repositories. Dependency/integration failures (19.5%) and data/type handling failures (17.6%) are the dominant root causes — precisely the surface the mutation boundary governs.[^4] The probabilistic-deterministic interface is the primary fault locus in agentic systems. Princeton's companion research found that reliability gains lag noticeably behind capability progress — 18 months of accuracy improvements yielded only modest reliability improvements.[^5] More capable agents are not proportionally more trustworthy. The reasoning layer's competence boundary does not expand as fast as its capability boundary — a more capable probabilistic caller is still a probabilistic caller.

---

## Lineage

The idea that mutations deserve special architectural treatment is not new. Five traditions arrived at overlapping conclusions — each solving a different problem, each contributing a primitive that Factor II inherits. What none of them faced was a probabilistic caller.

**CQRS** (Meyer 1988, Young 2010) established that reads and writes are categorically different operations that deserve different architectural treatment.[^6] Factor II inherits the asymmetry: hallucinated reads are generally recoverable; hallucinated writes are not. But the agentic context adds a nuance CQRS did not face — some reads share the governance requirement with writes. A read that surfaces regulated data into a probabilistic caller's context may be non-recoverable in the same sense a write is: the exposure cannot be undone. Factor II applies CQRS's scoping distinction but draws the boundary at governance need, not strictly at read vs. write.

**Event Sourcing** (Fowler 2005, Young 2010) demonstrated that separating what someone *intends* from what *actually happened* is productive, and that the gap between the two is where validation and business rules belong.[^7] Factor II puts governance in that same gap — between the reasoning layer's expressed intent and the mutation the control plane executes.

**The Saga pattern** (Garcia-Molina and Salem 1987, Vasters 2012) introduced compensation as a design-time architectural constraint: every forward operation ships with its reversal.[^8] Factor II inherits this directly — "compensation is defined before the operation begins." The new wrinkle: in the saga tradition, the coordinator has access to both forward and compensating transactions. In Factor II's world, the agent that triggers a charge may not have `refund` in its namespace. The compensation path exists in the system; the control plane routes it to an authorized actor. (Factor IV governs which callers can see which capabilities.)

**MRKL** (Karpas et al. 2022) was the first AI-era formalization of "the LLM decides what, deterministic code decides how."[^9] MRKL framed this as capability extension — the LLM lacks abilities, so give it expert modules. Factor II reframes it as capability confinement — the reasoning layer *has* capabilities (the Replit agent reached the database fine), but those capabilities are trustworthy only within the arena of reasoning about intent. The separation serves the enterprise systems' safety, not the reasoning layer's capability.

**The 2024–2026 convergence** confirms that multiple teams arrived at the same conclusion independently: HumanLayer, CrewAI, Google ADK, Anthropic, and Microsoft all built or recommended architectures where the reasoning layer expresses intent and deterministic infrastructure handles everything after.[^10] The Bhattarai/Vu paper from Los Alamos (February 2026) provides the formal argument: no training-only procedure can guarantee deterministic command-data separation in autoregressive transformers, so architectural enforcement is necessary.[^11] The Shah et al. fault taxonomy provides the empirical evidence: the probabilistic-deterministic interface is where agentic systems fail most often.[^4]

The thread connecting all five traditions: the boundary between intent and execution is where governance belongs. Factor II's contribution is applying that principle to a caller that may invent a command, repeat a command, misinterpret a result, or partially complete a saga and act as if it had finished — and extending the boundary to include reads whose exposure carries consequences equivalent to a write.

---

## Related Factors

This factor addressed how mutations enter the control plane — and which high-risk reads share the governance requirement. How governance scales across the mutation inventory — which mutations warrant heavier pre-execution controls, which can be lighter — is Factor I's consequence spectrum applied at the mutation boundary. For what the interface between caller and control plane looks like, see Factor III: Intent-Based Communication. For how the caller's tool namespace is scoped and governed, see Factor IV: Bounded Access. For how duplicate mutations are prevented when the caller retries, see Factor V: Safe Retries. For what the control plane returns to the caller when mutations fail, see Factor VI: Recovery Contracts. For how every mutation is recorded, see Factor VII: Structural Observability.

---

## Notes

[^1]: Replit AI coding agent incident, July 2025. Production database deletion during code freeze; 1,206 executives, 1,196 companies. Agent fabricated 4,000 fake records afterward. No architectural boundary between reasoning and state.

[^2]: Georgia Tech/Intel (2025). Analysis of agentic workload latency distribution: tool processing on CPUs accounts for 50–90% of total latency. The finding makes the performance cost of mediation concrete — and the case for consequence-proportional governance specific.

[^3]: OpenAI Operator incident, February 2025. Unauthorized $31.43 Instacart purchase when tasked with researching egg prices.

[^4]: Shah, D., et al. "Characterizing Faults in Agentic AI." arXiv:2603.06847, March 2026. 13,602 issues across 40 repositories; 385 faults coded into 37 categories. Validated by 145 practitioners (mean rating 3.97/5, 83.8% confirming coverage of faults they had encountered).

[^5]: Princeton SoDA Lab (2025–2026). "Towards a Science of AI Agent Reliability." Finding: 18 months of accuracy improvements yielded only modest reliability improvements. Reliability gains lag capability progress.

[^6]: Meyer, B. *Object-Oriented Software Construction*, 2nd ed. Prentice Hall, 1997 (1st ed. 1988). Young, G. "CQRS, Task Based UIs, Event Sourcing agh!" *codebetter.com*, Feb 2010. Young, G. *CQRS Documents*. cqrsinfo.com, Nov 2010.

[^7]: Fowler, M. "Event Sourcing." *martinfowler.com/eaaDev/EventSourcing.html*, 12 Dec 2005. Young, G. *CQRS Documents*, Nov 2010.

[^8]: Garcia-Molina, H. and Salem, K. "Sagas." *Proceedings of the 1987 ACM SIGMOD International Conference on Management of Data*, pp. 249–259. Vasters, C. "Sagas." MSDN blog, 1 Sep 2012; archived at vasters.com.

[^9]: Karpas, E. et al. "MRKL Systems: A modular, neuro-symbolic architecture that combines large language models, external knowledge sources and discrete reasoning." arXiv:2205.00445, May 2022.

[^10]: HumanLayer, *12-Factor Agents*, github.com/humanlayer/12-factor-agents (2024–2025). CrewAI Flows (Moura, 2025–2026). Google Agent Development Kit documentation (2025). Anthropic, "Building Effective Agents" (Dec 2024) and "Writing Tools for Agents" (2025). Microsoft, "Securing MCP" (April 2026).

[^11]: Bhattarai, B. and Vu, T. "Trustworthy Agentic AI Requires Deterministic Architectural Boundaries." arXiv:2602.09947, Los Alamos National Laboratory, February 2026. Preprint; not yet independently cited. The impossibility argument is logically strong but should be cited as supporting evidence, not foundational claim.
