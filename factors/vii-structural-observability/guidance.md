# Factor VII: Structural Observability — Extended Guidance

## Desired Outcome

When this factor is fully implemented, any authorized party can answer "what did the agent do?" from the record — at any point after the fact, without consulting application logs, interviewing developers, or reconstructing timelines from fragments. Every capability invoked against a system of record produces a structured record at invocation time, by the same control plane that performed the operation — captured to the extent an auditor would need, not to the extent a developer would stop caring. Incident response operates on evidence, not archaeology. Regulatory inquiries are answered from the record, not from investigation. The organization stops discovering what its agents did only after something goes wrong — and the control plane stops spending effort on post-hoc reconstruction.

---

## How This Works

The principle makes its boldest cross-factor claim: "When the control plane already enforces deterministic mutations (II), sequences operations with idempotency tokens (V), and returns structured error contracts (VI), the audit trail is not a separate concern — it is the emergent property of work the architecture already does." This section demonstrates where that claim holds, where it requires supplementation, and what a complete implementation looks like.

### What each factor contributes to the record

**Factor II provides the transaction boundary.** Every mutation passes through the control plane — it does not need to be "hooked into" the mutation path; it *is* the mutation path. The audit record's "which systems were affected" field falls out of what the control plane already knows from executing the operation.

**Factor III provides the intent semantics — and the structural presence for reads.** The declared intent that enters the control plane through an intent-aligned interface gives the audit record its "what was the declared intent" field. `check_out_guest` is auditable; `stripe_create_charge → update_salesforce_contact → update_pms_room_status` requires reconstruction. Factor III's intent boundary also puts the control plane in the path of *all* invocations — reads included — because every capability is invoked through an intent-aligned interface. This is what makes it structurally possible for the control plane to audit reads over regulated data even though Factor II's mutation guarantees do not apply to them.

**Factor V provides a correlation key for mutations.** The idempotency token tags each mutation uniquely — the same token that prevents duplicate execution also links the audit record to its operation. Reads are inherently idempotent and do not carry idempotency tokens. The audit record therefore needs its own primary key — an invocation ID assigned by the control plane at invocation time for every capability invoked regardless of type. For mutations, the idempotency token is a secondary field in the audit record, linking it to Factor V's deduplication logic. For reads, the invocation ID stands alone.

**Factor VI provides the outcome semantics.** Structured error contracts return business-semantic outcomes rather than HTTP status codes. The audit record's "what was the outcome" field is the error contract's result, persisted. For mutations, this includes partial success and compensation detail. For reads, the outcome is simpler — data returned, access denied, system unavailable — but still belongs in a structured result.

### Where the emergent property holds

Durable execution platforms provide the strongest evidence. Temporal persists a complete event history for every workflow execution — every command generates events that serve simultaneously as the crash-recovery mechanism and the audit log. Restate tracks every step of code execution in a journal, recording both operations and their results. Azure Durable Functions uses event sourcing: the append-only store records the full series of actions.[^1] Greg Young, who formalized event sourcing with CQRS, was originally attracted to the pattern because he needed auditing.[^2] When the execution model is event-sourced, the audit trail is a view over the event log — not a separate system.

### Where supplementation is required

The emergent trail is structurally complete for mutations but not legally hardened, and it does not cover all audit-qualifying events. Four gaps require intentional supplementation.

**Tamper-evidence.** Durable execution event histories are append-only for replay purposes but not cryptographically tamper-evident by default. Hash-chaining, Merkle proofs, or digital signatures must be layered on top.

**Authenticated identity.** The control plane knows which capability was invoked and by which workflow, but binding that invocation to an authenticated human or system identity requires integration with the identity layer.

**Retention and immutability.** Event histories are retained for operational replay. Regulatory retention (SOX: seven years; FINRA: WORM) requires a separate retention policy. The event store's operational lifecycle and the audit record's compliance lifecycle are different.

**Read-access audit in regulated domains.** Reads over regulated data — ePHI under HIPAA, cardholder data under PCI-DSS, personal data under GDPR — require audit records even though no mutation occurs. Factor II's *guarantees* (deterministic execution, compensation) apply to mutations, but Factor II itself acknowledges that high-risk reads share the governance requirement and should route through the control plane. The audit record for a read does not benefit from Factor V's idempotency token or Factor VI's compensation semantics — but it does benefit from the control plane's presence. Factor I's principle governs: in regulated contexts, the control plane's scope widens. It never narrows.

**A note on observability telemetry.** OpenTelemetry spans, metrics, and logs are a transport layer that can carry audit records, but OTel is neither the record nor its guarantees. OpenTelemetry's GenAI semantic conventions define spans for model operations and tool execution but do not address delivery guarantees, immutability, or actor attribution. Building an audit system on OTel without adding guaranteed delivery, content hashing, and immutable storage is building a compliance program on best-effort infrastructure.

### One proven pattern

Each capability invocation writes an audit record to an append-only store at invocation time. The schema is replaceable; the principle is not: five questions answered before anyone asks them. Two examples illustrate the structure — one mutation with partial failure, one regulated read.

A guest checkout touching three backend systems:

```json
{
  "invocation_id": "inv-20260406-a8f3c",
  "caller_identity": { "user": "front-desk@hotel.com", "auth": "oauth2-session-9x2k" },
  "declared_intent": { "tool": "check_out_guest", "params": { "booking_ref": "BK-4821" } },
  "operation_type": "mutation",
  "idempotency_token": "idem-4821-checkout-001",
  "affected_systems": [
    { "system": "payment-processor", "operation": "charge", "outcome": "success" },
    { "system": "crm", "operation": "update-contact", "outcome": "success" },
    { "system": "pms", "operation": "release-room", "outcome": "failed", "compensation": "retry-scheduled" }
  ],
  "business_outcome": { "status": "partial_success", "code": "ROOM_RELEASE_PENDING", "recovery": "retry" },
  "record_hash": "sha256:e3b0c44298fc1c14...",
  "previous_hash": "sha256:d7a8fbb307d78094..."
}
```

An agent accessing a patient's medication history:

```json
{
  "invocation_id": "inv-20260406-b2d7e",
  "caller_identity": { "user": "dr.chen@hospital.org", "auth": "smart-on-fhir-session-7k3m" },
  "declared_intent": { "tool": "get_patient_medications", "params": { "patient_mrn": "MRN-9920" } },
  "operation_type": "read",
  "affected_systems": [
    { "system": "ehr", "operation": "query-medications", "outcome": "success" }
  ],
  "business_outcome": { "status": "success", "records_returned": 12 },
  "record_hash": "sha256:4a2c8f90b1e64d37...",
  "previous_hash": "sha256:e3b0c44298fc1c14..."
}
```

The mutation record carries an `idempotency_token`; the read record does not. Neither record contains the sensitive data itself — the patient's medications do not appear in the audit record, and the credit card number processed during checkout does not appear either. The hash chain links records in sequence; an external anchor (a periodic Merkle root published to a separate store) provides third-party verifiability. The record is written synchronously with the operation — not asynchronously from a log pipeline.

---

## When This Applies (and When It Doesn't)

Factor VII applies to any system where AI agents take actions that must be accountable after the fact. In regulated industries — financial services, healthcare, legal, government — the audit trail is legally mandated. In multi-agent systems, attribution across agent boundaries makes structural observability essential for incident response. Two contexts change how the guidance applies, and one context narrows its scope.

**Developer and prototyping environments.** The full five-field audit record is premature for a prototype. But the habit of structured records prevents the most common failure: teams build without audit infrastructure, then attempt to bolt it on at production time — discovering that the control plane was never in the data path and the records cannot be retroactively produced. Even in prototyping, using a structured result envelope (Factor VI) gives the team something to project an audit view over when the system matures.

**Internal, non-regulated use cases.** Structured records remain valuable for incident response, but the compliance requirements — retention duration, WORM storage, tamper-evidence — may not apply. The guidance helps teams distinguish which fields are architecturally necessary (declared intent, affected systems, outcome) from which are regulatorily necessary (authenticated identity, hash-chaining, retention policy).

**Actions within the reasoning layer only.** When a probabilistic caller answers a question from its training data, performs a calculation, or reasons over content already in its context window, no capability is invoked against a system of record. There is nothing for the control plane to audit because there is no control plane action. However: a probabilistic caller that *reads* regulated data — even without mutating it — crosses the audit boundary. An agent summarizing a patient's medication history is accessing ePHI, and HIPAA requires that access be logged. Mutations cross the audit boundary by default. The less obvious test: did the caller access data that a regulatory framework requires you to track, even if nothing was changed?

### The audit boundary as a design-time decision

The audit boundary — which operations qualify for audit-grade records — is a design-time decision that widens with regulatory context, per Factor I's principle that the governance dial turns in one direction.

At the narrowest boundary (prototyping), no operations cross the audit boundary. Developer traces suffice. In production without regulatory constraints, mutations cross the audit boundary; reads may be tracked via developer traces for incident response but do not require audit-grade records. In regulated production, mutations *and* reads over regulated data cross the boundary. Factor II already acknowledges this widening — high-risk reads that share the governance requirement should route through the control plane even though Factor II's deterministic execution and compensation guarantees do not apply to them. Factor III makes this structurally possible — all invocations, reads included, enter through intent-aligned interfaces and therefore pass through the control plane. Factor VII's contribution is ensuring that the control plane's presence for these reads produces an audit record, not just a developer trace.

### Capture extent: the auditor's trace, not the developer's

The audit boundary determines *which* operations produce audit records. Capture extent determines *how deeply* each record traces the operation's consequences. A developer trace for a checkout might capture "called Stripe, got 200." An auditor needs the full consequence chain: Stripe charged, CRM updated, PMS failed, compensation scheduled. The developer's trace is correct for debugging — it stops at the boundary where debugging ends and accountability begins.

The control plane can only capture what it orchestrated and what it received back. Consequences that occur inside a downstream system — a payment processor's internal fraud check, a CRM's cascade triggers — are outside the control plane's observability scope. The design question is whether the control plane is in the path of enough of the operation to produce a record at auditor-grade extent. If it is not, that is a Factor II concern (widen the control plane's transaction boundary) or a Factor III concern (the interface returns too little information about what happened downstream). Factor VII takes what the control plane can observe and persists it at the extent the auditor requires — but it cannot persist what the architecture never made visible.

---

## Diagnostic Framework

The industry uses "logging," "tracing," "observability," and "audit" interchangeably. The conflation is actively harmful. Every agentic framework provides some form of tracing — LangSmith, OpenAI traces, AWS Bedrock traces — capturing what the developer needs: token counts, latency, error messages, reasoning chain steps, model version. These are *developer traces*: retention measured in days, probabilistic sampling acceptable, delivery best-effort. None of these properties is compatible with accountability. An *audit record* captures what the regulator, auditor, or incident responder needs: authenticated identity, declared intent, affected systems, business-semantic outcome, tamper-evidence. Retention is measured in years. Capture is 100% of boundary-qualifying events. Delivery must be guaranteed. Microsoft 365 Copilot demonstrated the cost of conflating the two: the system produced developer-grade telemetry for over a year while silently failing to generate audit-grade records in Microsoft Purview — organizations relying on Purview for HIPAA, SOX, or GDPR compliance had incomplete historical logs and were never informed.[^11]

The principle names five questions every capability invocation must answer: who requested this, what was the declared intent, which systems were affected, what was the outcome, and can the record prove it has not been altered. The extended guidance formalizes these into five diagnostic dimensions, ordered from easiest to spot to hardest to get right.

**Dimension 1: Record completeness.** *Does every audit-qualifying capability invocation produce a structured audit record?*

Pick any invocation from the last 24 hours. Can you retrieve its audit record within 60 seconds? Are there invocations that produce no record at all — the agent acted but no event was emitted? Are there invocations that produce developer traces but not audit records? If the answer is "no record exists," nothing else matters.

**Dimension 2: Field adequacy.** *Does the record carry the five fields?*

For a given audit record, can you answer: who requested this, what was the declared intent, which systems were affected, what was the business-semantic outcome, and can the record prove it has not been altered? Missing fields follow a consistent pattern: authenticated identity is frequently absent (the record shows which API key, not which human). Business-semantic outcome is frequently replaced by HTTP status code. Tamper-evidence is almost never present.

**Dimension 3: Production timing.** *Is the record produced at invocation time by the control plane, or reconstructed after the fact?*

If the logging pipeline fails for one hour, do you lose audit records for operations that completed successfully? Any operation that loses its record when the pipeline fails has a reconstructed timeline, not a structural one. Asynchronous logging pipelines that can drop records, nightly batch jobs that reconstruct timelines from application logs, ETL processes that assemble audit records from multiple sources — all fail this dimension.

**Dimension 4: Tamper-evidence and retention.** *Can the record prove it has not been altered? Is retention matched to regulatory requirements?*

If an administrator modified an audit record from six months ago, would you detect the modification? Records stored in mutable databases without hash-chaining or Merkle proofs fail this dimension. Most teams do not have tamper-evidence — teams with Dimensions 1–3 are ahead of most of the industry. But regulatory requirements are converging: SOX mandates seven-year retention with criminal penalties for alteration; FINRA requires WORM storage; the EU AI Act requires automatic recording with six-month minimum retention.

**Dimension 5: Control plane coverage.** *Does the control plane see enough of the end-to-end operation to produce a record worth having?*

This is the hardest dimension because it requires tracing the full operation, not inspecting the record. Pick a multi-system operation — a checkout, a coverage determination, a trade execution. Follow it from the reasoning layer's declared intent through every system of record touched. At each step, ask: is the control plane in the path? Does it see the request, the downstream system's response, and the business-semantic outcome? Where work happens outside the control plane — a downstream service that makes its own backend calls, an agent-to-agent delegation where the downstream agent has unmediated access, a tool that shells out to a subprocess — the audit record has a blind spot. The record cannot show what was never visible to the architecture.

A team can pass Dimensions 1–4 and still have consequential work happening outside the control plane's scope. The checkout record says "called payment processor, got success" — but the CRM update and the PMS room release happened in a downstream service the control plane didn't orchestrate. The record is complete for what the control plane saw. It is incomplete for what actually happened. This is where Factor VII intersects with Factor II (widen the control plane's transaction boundary to include those downstream systems) and Factor III (the interface should return richer information about what happened downstream). Factor VII's diagnostic surfaces the gap; the fix may live in another factor's territory.

---

## Named Anti-Patterns

Five failure modes appear repeatedly in production systems, one per diagnostic dimension.

**1. The silent agent** *(Dimension 1: Record completeness).* The agent takes actions but no audit event is emitted. Microsoft 365 Copilot could access and summarize enterprise files without generating entries in Purview audit logs — a vulnerability that persisted for over a year before a quiet server-side fix. The structural cause: Copilot's document retrieval was decoupled from audit event emission. Detection: compare invocation counts from the orchestration layer against audit record counts. A mismatch means you have silent agents.

**2. The status-code audit** *(Dimension 2: Field adequacy).* The audit record exists but carries only what the developer needed: timestamp, tool name, HTTP status code, latency. An audit record that says "200 OK" for a checkout that partially failed communicates none of the business-semantic distinctions — the Factor VII manifestation of the Factor VI problem. Google's Factor XV names the structural cause: AI applications can return HTTP 200 with useless or incorrect answers.[^8] Detection: review a sample of audit records and attempt to answer the five questions. Fields you cannot answer are fields you do not have.

**3. The reconstructed timeline** *(Dimension 3: Production timing).* Audit records are assembled after the fact from application logs, ETL pipelines, or nightly batch jobs. J.P. Morgan's $200 million CFTC settlement (2024) revealed that billions of order messages went unsurveilled across 30+ trading venues — data existed across systems but surveillance tools were not configured to capture it.[^9] The 2010 Flash Crash took regulators five months to reconstruct 36 minutes of trading activity because no unified audit trail existed across exchanges.[^10] Detection: ask "if the logging pipeline fails for one hour, which operations lose their audit records?" Every operation in that set has a reconstructed timeline.

**4. The mutable ledger** *(Dimension 4: Tamper-evidence).* Audit records are stored in a database that administrators can modify. No hash-chaining, no Merkle proofs, no WORM. When the entity that controls the logs is also the entity under investigation, the record's integrity depends on trust rather than architecture. AWS QLDB — a purpose-built immutable ledger database — was retired in July 2025 without a formal announcement, demonstrating that even vendor-provided immutability guarantees can be withdrawn. Detection: could a database administrator alter a six-month-old audit record without detection? If yes, you have a mutable ledger.

**5. The partial picture** *(Dimension 5: Control plane coverage).* The audit records are complete, well-fielded, produced at invocation time, and tamper-evident — for the operations the control plane orchestrates. But consequential work happens outside the control plane's scope. A downstream service makes its own backend calls that the control plane never sees. An agent-to-agent delegation crosses into a system where the downstream agent has unmediated access. A tool invokes a subprocess that touches a system of record directly. The audit trail is architecturally sound for the operations inside the boundary and structurally blind to the operations outside it. Detection: trace a representative multi-system operation end-to-end. At each point where a system of record is touched, ask: does the control plane see this? Every step it doesn't see is a blind spot in the audit record — and a gap that no amount of record-quality improvement can fix.

---

## Industry Examples

Three domains illustrate how the same audit record architecture adapts to different backend systems, regulatory environments, and consequence severities — and how the reasoning layer / control plane boundary determines what the record can capture.

### Hospitality: the partial-failure checkout

A guest checks out of Dewy Resort. The reasoning layer declares `check_out_guest` with the booking reference. The control plane orchestrates three backend operations: Stripe processes the final charge, Salesforce updates the guest contact record, and the PMS releases the room. Stripe succeeds. Salesforce succeeds. The PMS call fails.

Without Factor VII, the developer trace captures what the framework saw: a tool call, a latency measurement, and an error code. An incident responder investigating the next day cannot determine from the trace which backend systems succeeded, which failed, or whether compensation was initiated. The guest calls about an unexpected charge; the support team starts reconstructing from Stripe logs, Salesforce activity history, and PMS error logs — three systems, three log formats, no unified record.

With Factor VII, the control plane produced an audit record at invocation time carrying the caller's identity, the declared intent (`check_out_guest`, booking BK-4821), each backend system's operation-specific outcome (Stripe: charge success, Salesforce: update success, PMS: room release failed, compensation: retry scheduled), and the business-semantic result (partial success, room release pending). The incident responder queries one record. The five questions are answered before anyone asks them. The partial failure is visible — not because someone logged it, but because the control plane recorded what it orchestrated.

### Healthcare: the regulated read

A clinical agent retrieves a patient's medication history to prepare a discharge summary. The reasoning layer declares `get_patient_medications` with the patient's medical record number. The control plane queries the EHR system and returns the result.

No mutation occurred. No state changed. Factor II's deterministic execution guarantees do not apply. But the patient's medication history is electronic protected health information under HIPAA, and 164.312(b) requires that access be logged — who accessed it, when, through what authorization, and what was returned.

Without Factor VII, the developer trace captures a successful tool call with a latency measurement. The compliance team running a quarterly HIPAA audit cannot determine from the trace which clinician's authorization was used, whether the access was clinically justified, or how many patient records the agent accessed that day. The trace was designed for debugging, not for compliance.

With Factor VII, the control plane produced an audit record carrying the authenticated clinician's identity (Dr. Chen, SMART on FHIR session), the declared intent (`get_patient_medications`, patient MRN-9920), the system accessed (EHR, query-medications, success), and a summary-level outcome (12 records returned — not the medication data itself). The compliance team queries the audit store for all ePHI access events in the period. The record exists because the control plane was in the path of the read — Factor III's intent-aligned interface put it there — and Factor VII ensured the control plane's presence produced an accountability-grade record, not just a pass-through.

### Financial services: the multi-venue trade

A trading agent submits a buy order. The control plane routes to a matching engine, receives an acknowledgment delay, and the reasoning layer — optimized for helpfulness — re-expresses the intent through a different order path. Factor V's deduplication prevents the duplicate execution. But what did the auditor see?

Without Factor VII, the developer trace shows two tool calls and one deduplicated result. A compliance officer reviewing the agent's activity cannot determine from the trace why the second submission occurred, whether the agent was retrying or initiating a new order, or what the original submission's parameters were. The deduplication worked — no duplicate trade — but the *reasoning* behind the duplicate submission is invisible.

With Factor VII, both invocations produced audit records at invocation time. The first carries the declared intent, the venue, and the outcome (acknowledgment delayed). The second carries its own declared intent, the venue, and the outcome (deduplicated by Factor V — original result returned). The correlation between them is visible through the idempotency token (a secondary field linking both to the same deduplication key). The compliance officer can reconstruct the sequence: original submission, timeout, retry through a different path, deduplication prevented duplicate execution. The agent's behavior is attributable and bounded — not because the agent explained itself, but because the control plane recorded each invocation as it happened.

---

## Tradeoffs and Tensions

**Record everything vs. minimize what you store.** GDPR Article 5(1)(c) requires data minimization. HIPAA's minimum necessary rule restricts PHI access. PCI-DSS scope minimization reduces audit burden. Meanwhile, SOX requires seven-year retention, FINRA requires WORM storage, and the EU AI Act requires automatic recording. The tension is real but architecturally resolvable. Tokenization at the boundary replaces sensitive values with non-reversible tokens before they enter the audit pipeline; PCI DSS 4.0 validates this pattern. Tiered logging applies differential retention: a hot tier (0–90 days) holds pseudonymized traces for active investigation, a warm tier (3–12 months) retains reduced-detail records, and a cold tier stores aggregated records for long-term archival. Gateway-level redaction detects PII in agent requests and responses in real time, replacing it with compliant surrogates before the data reaches the audit store. The audit record captures *what happened* without capturing *what sensitive data was involved*.

**Synchronous write vs. performance.** Writing the audit record synchronously with the operation adds latency. For high-throughput systems, this may be unacceptable. The pragmatic resolution: write to a local append-only buffer synchronously (fast), then replicate to the durable store asynchronously (reliable). The local buffer provides the "produced at invocation time" guarantee; the replication provides durability.

**Vendor-dependent immutability vs. architectural independence.** AWS QLDB's retirement in July 2025 is the cautionary tale — teams that relied on a single vendor for immutability guarantees lost them without a formal announcement. Architectural independence requires hash-chaining owned by the control plane rather than the storage vendor, Merkle roots anchored to a separate system, and the ability to migrate the audit store without losing tamper-evidence.

**Completeness vs. volume.** 100% capture for audit records generates volume that scales linearly with agent activity. The resolution is the core distinction between developer traces and audit records: developer traces can be sampled; audit records cannot. Separating the two pipelines means different sampling, retention, and storage policies apply to each. The audit pipeline is smaller (only events that cross the audit boundary) and more expensive per record (guaranteed delivery, immutable storage). The developer trace pipeline is larger and cheaper.

**Capture extent vs. control plane scope.** The auditor's needs may extend beyond what the control plane can observe. A downstream system's internal side effects, a third-party API's cascade triggers, a partner service's internal state changes — all may be consequential but invisible to the control plane. The resolution is not to expand the audit record's claims beyond the control plane's observability — that creates a record that implies completeness it cannot guarantee. Instead, the record captures what the control plane orchestrated and received back, and the audit boundary design (§ 3) ensures the control plane is in the path of enough of the operation to produce records at the extent the auditor requires. Where it is not, that is an architectural gap to address through Factor II (widen the transaction boundary) or Factor III (the interface returns richer downstream information) — not a gap to paper over in the audit record.

---

## The Cost of Unobservable Agents

**Incident response becomes archaeology.** The Flash Crash took five months to reconstruct 36 minutes. Without structural observability, every incident investigation starts from zero — assembling fragments from application logs, database snapshots, and developer recollection. The investigation cost scales with reconstruction difficulty, not with incident severity.

**Regulatory exposure scales with agent autonomy.** J.P. Morgan: $200 million for unsurveilled trading messages. Two Sigma: $90 million for unaudited model changes. UnitedHealth: federal court-ordered disclosure because the algorithm's decisions were not observable. Each enforcement action was brought under existing law — not under future AI-specific legislation. As agents take more autonomous actions, the number of unauditable decisions multiplies, and each is potential regulatory exposure under frameworks that already exist.

**Compliance becomes impossible at scale.** At enterprise scale — thousands of agents making millions of invocations per day — manual audit is impossible. The audit trail must be architectural or it does not exist. Microsoft Copilot demonstrated the failure mode: organizations relying on Purview for compliance had incomplete historical logs for over a year and were never informed. The gap was not a configuration error. It was a structural absence.

**Reconstruction from scattered traces fails for probabilistic callers.** Traditional systems tolerate developer traces living across dashboards, vendor consoles, and log aggregators — consolidated on demand when an engineer investigates. For deterministic callers, this is expensive but feasible: the caller's behavior is reproducible, so fragments can be reassembled. For probabilistic callers, the fragments cannot be reassembled — the reasoning that connected them is non-reproducible, context windows have overflowed, and the same intent was expressed differently each time. The "reconstruct when we need it" assumption that is tolerable for deterministic systems is an accountability gap for probabilistic ones. This is the structural argument for audit records over developer traces: not that developer traces are bad, but that they cannot serve as a fallback reconstruction source when the caller's behavior cannot be replayed.

---

## Lineage

Factor VII draws from four independent traditions, each solving a piece of the problem. None faced the challenge this framework addresses: a probabilistic caller operating against enterprise systems of record.

**Audit as a byproduct of correct execution.** Greg Young, who formalized event sourcing with CQRS, was originally attracted to the pattern because he needed auditing.[^2] Martin Fowler captured the core principle: every state change is initiated by an event object, and the audit trail is a view over the event store rather than a separate system.[^3] This is Factor VII's starting point — when the control plane records every step for crash recovery, the audit trail falls out of work the architecture already does. But event sourcing assumed a deterministic caller whose behavior could be replayed from the event log. A probabilistic caller's behavior cannot be replayed. And event sourcing addressed writes; regulatory frameworks like HIPAA demand that the record extend to reads over regulated data. Factor VII inherits the principle and widens its scope.

**Proving the record has not been altered.** Event sourcing produces append-only logs, but append-only is not tamper-evident. Crosby and Wallach demonstrated that Merkle tree structures can prove any single entry's integrity in a log of a billion entries.[^4] Certificate Transparency (RFC 6962) proved this pattern works at internet scale.[^5] Factor VII requires this supplementation as a first-class architectural commitment — not an optional hardening step — because when the entity controlling the logs may be the entity under investigation, the record's integrity cannot depend on trust.

**What the record must contain and how long it must last.** Enterprise audit trail standards provided the field requirements and the retention mandates. PCI-DSS Requirement 10.2 mandates automatic audit trails with specific fields per event — the longest-running enterprise audit trail standard.[^6] SOX Section 802 mandates seven-year retention with criminal penalties for alteration. HIPAA 164.312(b) extends the audit requirement to access events — reads, not just writes. Factor VII's read-access audit requirement descends directly from HIPAA's extension: when a probabilistic caller accesses regulated data, the governance requirement does not distinguish between reads and writes.

**The record must be produced by the system that did the work.** The twelve-factor app's Factor XII (Admin Processes) established the principle that ancillary work must share the same architectural substrate as primary work.[^7] For Factor VII, this means audit records must be produced by the same control plane that performs operations — not reconstructed afterward from a logging pipeline or a batch job. Google's Factor XV identified why this matters more for AI systems: a probabilistic caller can return a successful response while producing a wrong answer, making status codes structurally insufficient as audit evidence.[^8] The audit record must capture business-semantic outcomes — "the checkout partially completed with compensation pending," not "200 OK" — because an auditor cannot replay the probabilistic caller's reasoning to determine what actually happened.

[^1]: Microsoft. "Azure Durable Functions: Orchestrations." Azure Documentation.

[^2]: Young, G. (2010). "CQRS Documents." https://cqrs.files.wordpress.com/2010/11/cqrs_documents.pdf

[^3]: Fowler, M. (2005). "Event Sourcing." https://martinfowler.com/eaaDev/EventSourcing.html

[^4]: Crosby, S.A. & Wallach, D.S. (2009). "Efficient Data Structures for Tamper-Evident Logging." *USENIX Security Symposium.*

[^5]: Laurie, B., Langley, A., & Kasper, E. (2013). "Certificate Transparency." RFC 6962. https://www.rfc-editor.org/rfc/rfc6962

[^6]: PCI Security Standards Council. (2022). *PCI DSS v4.0.* Requirement 10.2.

[^7]: Wiggins, A. (2011). "The Twelve-Factor App: XII. Admin Processes." https://12factor.net/admin-processes

[^8]: Gupta, K.K. (2025). "Rethinking the Twelve-Factor App framework for AI." Google Cloud Blog. https://cloud.google.com/transform/from-the-twelve-to-sixteen-factor-app

[^9]: U.S. Commodity Futures Trading Commission. "CFTC Orders JPMorgan to Pay $200 Million." Press release, May 2024.

[^10]: U.S. Securities and Exchange Commission & CFTC. "Findings Regarding the Market Events of May 6, 2010." Joint report, September 2010.

[^11]: Bargury, M. (2024). Black Hat USA presentation on Microsoft 365 Copilot audit log gaps, August 2024. Korman, Z. (2025). Independent confirmation of persistent vulnerability, July 2025. Microsoft applied server-side fix August 17, 2025.

---

## Related Factors

This factor addressed what the control plane records — the structured evidence of every action taken. For how the governance dial determines the control plane's scope, see Factor I: Governed Operations. For how mutations enter the control plane, see Factor II: Deterministic Mutations. For how intent-aligned interface composition shapes what the audit record can capture, see Factor III: Intent-Based Communication. For how the namespace determines which capabilities exist, see Factor IV: Bounded Access. For how the control plane makes operations safe to retry — and how the idempotency token serves as a secondary audit field for mutations — see Factor V: Safe Retries. For what the control plane returns to the caller, see Factor VI: Recovery Contracts.
