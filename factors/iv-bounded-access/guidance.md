# Factor IV: Bounded Access — Extended Guidance

## Desired Outcome

When this factor is fully implemented, each caller's namespace is materialized per call, scoped to the caller's role — and each tool within that namespace encodes its own authorization boundary in its execution context. What the agent can see is bounded by which tools exist in its runtime. What each tool can reach is bounded by whose authorization the execution context carries. A compromised agent — whether through prompt injection, confused retry, or adversarial tool metadata — cannot exceed the namespace because the namespace is all that exists in its runtime, and cannot exceed the execution context because the context is set by infrastructure, not by the agent. The organization can answer, for any caller: exactly which tools are in this caller's namespace, exactly what authorization boundary each tool carries, and exactly what the gap is between the governed namespace and the effective reach. Access control is a property of the control plane, not a behavior the agent is instructed to follow.

---

## How This Works

Every agent that can invoke a tool has a namespace and an execution context for each tool it invokes — whether or not the team has made a deliberate decision about either. The question is not whether these exist but whether the team is governing them, and how deeply.

The reason these are control plane responsibilities — and not the agent's — follows from a sixty-year-old problem in access control. When a program holds authority and exercises it on behalf of a caller, it is a *deputy* — and deputies can be confused. Hardy's 1988 paper named the problem; every tradition since — capabilities, least privilege, zero trust — assumed the deputy role was a given and asked how to make deputies less confusable. Better scoping. Better token management. Better policy evaluation. The entire posture is: the deputy holds authority, so help it hold authority correctly.

A probabilistic caller breaks this posture. It is optimized for helpfulness over accuracy — its default response to ambiguity is to try harder, not to stop.[^1] It cannot reliably distinguish whose authority it is exercising, not as a bug to be fixed but as a property of the reasoning layer. The correct response is not better confusion-proofing. It is to remove the agent from the deputy role entirely. The agent expresses intent. The control plane holds authority and determines what that intent is allowed to reach. No deputy, no confusion. This inversion — from "make the deputy smarter" to "eliminate the deputy" — is the architectural commitment that makes namespace governance and execution-context scoping control plane responsibilities rather than agent responsibilities.

### Namespace materialization

Every agent call begins with a set of capabilities available to the agent — its namespace. This includes registered tools and skills, but also CLI access, file system access, web search, code execution environments, agent-to-agent delegation endpoints, and any other invokable or executable surface the agent can reach. The namespace is composed by something: a static tool list in application code, a gateway filter applied per caller, a server instance that exposes only certain operations, or simply everything registered on every connected server plus whatever non-tool capabilities the runtime provides. *Namespace materialization* is the composition of that set. It is a control plane responsibility — it occurs before the model sees what's available, and it is performed by infrastructure, not by the agent. It is not tool discovery (the agent querying what's available), not tool filtering (removing items from a response), and not permission checking (validating at invocation time). Discovery is the agent asking; filtering is the gateway answering selectively; permission checking is the backend validating at call time; materialization is the set of capabilities made available by infrastructure. The distinction matters because teams conflate these four operations and each provides a different strength of guarantee.

### Execution-context scoping

Every capability in the namespace runs within an execution context that determines what it can reach — which records it can query, which operations it can perform, which systems it can touch. A tool invocation, a CLI command, a file system operation, an agent-to-agent delegation — each executes under some authorization boundary. That boundary is set by something: propagated credentials, a shared service account, a hardcoded API key, an OAuth token. *Execution-context scoping* is the practice of ensuring that boundary reflects the correct authorization for a transaction rather than a broad or static default. This is control plane attenuation — not a reasoning-layer guardrail, not a prompt instruction, not a runtime permission check. The authorization scope travels with the execution context, not with the agent's identity or instructions.

### Axis 1: Namespace governance

Every agent has two namespaces. The *primary namespace* is the set of capabilities the team deliberately materializes for a given call — the front door. The *effective namespace* is everything the agent can actually access during the transaction: the primary capabilities, their descriptions, their error responses, any additional invokable surfaces (CLI access, web search, file system, code execution, agent-to-agent delegation), and any internal services those capabilities can reach. Governing the namespace means controlling both.

**Level 0: Universal namespace.** The agent sees everything registered on every server it connects to. The primary namespace is whatever happens to be there, and the effective namespace is unknowably larger — every tool description, every error response, every non-tool capability the agent can reach. This is the state the factor describes: the full namespace, with the system prompt or the agent's judgment as the only constraint.

**Level 1: Hardcoded registry.** A developer has hardcoded a tool list in the application code. The primary namespace is bounded but does not vary by caller, role, or session. It is correct only as long as the hardcoded list matches the authorization requirements of every caller who reaches that server — and nothing validates that it does. The effective namespace remains unaudited.

**Level 2: Policy-based namespace generation.** The primary namespace is generated per caller through a policy mechanism — gateway-level filtering (Kong Tool ACLs, Cloudflare WorkOS integration, MCP gateway interceptors), per-call list construction based on caller identity, or Cedar policies evaluated per request. Different callers see different tool sets, and the namespace can change mid-session when authorization state changes. But the tools still exist on the upstream server — they are outside the primary namespace but inside the effective namespace. The boundary is enforced by policy, not by architecture. A discovery bypass, a server-side misconfiguration, or a prompt injection naming the tool directly can cross it. The guarantee is policy-based: the agent cannot see the tool, but the tool is still there.

**Level 3: Structural namespace separation.** Different callers are routed to entirely different server instances. The guest agent's MCP server exposes `search_bookings_on_behalf_of_guest` and `submit_service_request`. The staff agent's MCP server exposes `override_rate`, `comp_room`, and `view_all_guests`. Capabilities that do not exist in the caller's runtime cannot be reached regardless of policy state, bypass attempts, or agent behavior. The guarantee is existential: there is no tool to bypass to. When authorization state changes — a guest checks out and loses booking-modification rights, a staff member's role is revoked — the routing layer redirects subsequent calls to a different server instance or disconnects the session. The dynamism is in the routing, not in the filtering. Structural separation tightens the effective namespace significantly — tools that don't exist can't be reached — but the effective namespace still includes tool descriptions, error responses, and non-tool capabilities that Axis 2 governs.

### Axis 2: Transaction-depth governance

Namespace governance addresses which tools the agent can see and reach. Transaction-depth governance addresses what happens when a tool is invoked — what it can reach, what it reveals, and whether its behavior stays within the transaction's authorization boundary. This axis is independent: a team can be at Level 3 on the namespace axis and completely ungoverned on transaction depth.

**Level 0: Ungoverned.** Tools run with whatever credentials they were deployed with — design-time defaults, hardcoded API keys, pass-through service accounts. No one has examined what the execution context permits. The tools are black boxes: the team knows what they're called but not what they can reach.

**Level 1: Shared credentials, known.** The team knows the tools use a shared service account and understands what that account can access. They've accepted it as a shortcut but haven't scoped execution contexts per transaction. The risk is visible but unmitigated — every tool invocation carries the same broad credential regardless of who initiated it.

**Level 2: Verified caller access.** Each tool carries the correct authorization boundary for the transaction into its execution context — the named principal's OAuth scope, Salesforce profile, or RBAC role. The execution context bounds what each tool can reach when invoked. MiniScope formalizes this: it reconstructs permission hierarchies from each service's existing OAuth scopes and attaches the minimal scope set to each tool invocation.[^2]

The guest-facing `search_bookings_on_behalf_of_guest` tool can only return the authenticated guest's own records because the execution context carries the guest's Salesforce profile. The staff-facing `search_bookings_on_behalf_of_staff` tool can return any guest's records because the execution context carries a staff-level profile. The naming is deliberate: encoding the authorization scope in the tool name makes the boundary visible to developers, prevents accidental cross-wiring, and documents who the tool serves. A shared name like `search_bookings` across both namespaces obscures the boundary — a developer cannot tell from the name alone which authorization context it carries. Naming that follows Factor III's intent-aligned principles should encode authorization scope when different callers use different authorization boundaries for semantically similar operations.

**Level 3: Contracted execution boundaries.** Execution contexts are scoped to the named principal (Level 2), and the tool's own behavior is bounded. Internal service calls are traced and constrained — a tool cannot escalate to services outside the transaction's authorization boundary. Error responses are contracted and do not leak system architecture, credentials, or internal paths (Factor VI). Tool descriptions are reviewed for information leakage — they shape the agent's behavior and can be crafted to hijack tool selection (Factor III). The response path is also bounded: what the tool returns to the reasoning layer does not include system identifiers, internal state, or backend details that the intent-aligned interface correctly withheld on the way in.

### Where both axes converge

When both axes are fully governed — the namespace is structurally separated (Axis 1, Level 3) and transaction depth reaches contracted execution boundaries (Axis 2, Level 3) — the effective namespace converges with the primary namespace. What the agent can reach is what the team intended it to reach. Non-tool capabilities — CLI access, web search, file system access, code execution environments — are enumerated and bounded as part of that convergence: they belong to the effective namespace, and governing them requires attention to both axes.

---

## When This Applies (and When It Doesn't)

**Every agent that can invoke a capability.** Namespace materialization and execution-context scoping are present in every agent that can invoke capabilities. The governance spectrum applies to all of them. The levels describe how deliberately the team governs what is already there — not whether governance is needed.

**Agent-to-agent delegation.** The governance spectrum applies regardless of whether the caller is a human or another agent. When an A2A request arrives from an upstream agent, the control plane must evaluate two authorization boundaries, not one: the principal's (the human or organizational role on whose behalf the chain ultimately acts) and the delegating agent's own authorization to handle that class of data or operation. The effective authorization is the intersection. A guest principal may be authorized to view their own payment methods, but a third-party travel-planning agent delegated by that guest should not inherit payment-data access unless the agent itself is authorized to handle PCI-scoped data. The reverse also holds: a compliance-monitoring agent may be authorized to access audit logs across all accounts, but if the requesting principal is a line-of-business manager, the result set should be scoped to that manager's accounts — the agent's broad authorization does not elevate the principal's reach. Without this intersection, delegation becomes a capability-laundering mechanism in either direction — agents inherit principal authorization they shouldn't handle, or principals gain agent authorization they shouldn't reach. A2A's Extended Agent Cards advertise different capabilities to different authenticated callers, which is namespace materialization at the protocol level. For high-consequence systems, compound enforcement adds declarative policy evaluation through Cedar or Cerbos within each structurally separated namespace.

**Development vs. production.** Early-stage development often legitimately exposes the full namespace to accelerate iteration. The risk calculus changes when the agent touches production data or operates on behalf of real users. That transition — development to production — is where namespace governance must be introduced. Deferring governance is a legitimate choice during prototyping; the cost of retrofitting it after launch is higher than introducing it at the production boundary.

---

## Diagnostic Framework

Five diagnostic dimensions, ordered from easiest to hardest to spot. The first requires no tooling. The last requires understanding what the agent can reach that nobody authorized.

**D1: Exposure survival.** "If the system prompt and tool list were published today, what access control properties would be compromised?" This is the factor's litmus test — a thought experiment that requires no tooling and no infrastructure inspection. The healthy answer: "none — access control is enforced by the namespace and execution context, not by prompt instructions." The unhealthy answer reveals that the system prompt contains authorization logic: tool restrictions, conditional access rules, identity verification instructions. If publishing the prompt changes the security posture, access control is in the wrong layer. Start here.

**D2: Caller-sensitivity.** "Does the namespace change based on who the caller is?" The healthy answer names the mechanism by which different callers get different namespaces — gateway policy, server-per-role architecture, per-call composition logic. The answer that reveals Level 1: "every caller sees the same tool list" in a system with multiple roles or authorization boundaries.

**D3: Authorization-scope propagation.** "When a tool is invoked, how does the execution context reflect the authorization boundary for this transaction — and does that boundary hold on the return path?" The healthy answer names a specific mechanism — an OAuth scope, Salesforce profile, or RBAC role propagated into the execution context — and confirms that the response is bounded by the same scope. The answer that reveals the service-account deputy: "the tool uses a service account" or "the tool has its own credentials." The response-path check matters because a tool that correctly scopes its *query* but returns unbounded results to the reasoning layer has a Dimension 3 failure on the return path — the authorization boundary held on the way in but leaked on the way out.

**D4: Primary namespace.** "What capabilities are in this agent's namespace right now, and why is each one there?" The healthy answer is a complete enumeration — the team can list every capability materialized for this call and explain its presence. The unhealthy answer is uncertainty: "it connects to these servers, so... whatever's on them?" This is the front door — the namespace the team thinks about.

**D5: Effective namespace.** "What is the full composition of everything the agent can access during this transaction?" The primary namespace (D4) is the tools the team deliberately materialized. The effective namespace is everything the agent can actually touch, read, or be influenced by: tool descriptions that shape the agent's behavior and can be crafted to hijack tool selection; implementation details leaked through tool names or error responses that reveal system architecture the agent wasn't meant to see; capabilities beyond formally registered tools — CLI access, web search, file system access, code execution environments; internal service calls or workflow triggers within tools that reach resources outside the primary namespace. The gap between primary and effective namespace is where most real vulnerabilities live. The team may govern the front door precisely and still have an effective namespace far larger than they intended.

---

## Named Anti-Patterns

Each anti-pattern maps to a diagnostic — the failure that diagnostic is designed to detect. Ordered from easiest to hardest to spot.

**Prompt-as-ACL (access control list).**

*Diagnostic: D1 (Exposure survival).*

An access control list is a table that tells a system which subjects can perform which operations on which resources — enforced by the operating system or middleware, not by the subject itself. Prompt-as-ACL is what happens when that enforcement moves into the system prompt instead. The system prompt contains sentences restricting which tools the agent should use or under what conditions — "only use the refund tool when the customer has provided a valid order ID" or "verify the user's identity before accessing sensitive records." The authorization decision sits inside the same token stream that processes untrusted input. The system prompt that restricts tool usage and the injected instruction that overrides it are processed by the same model in the same pass. This is why probabilistic callers cannot be deputies: they cannot distinguish instructions from data — the confused deputy pattern manifests by design, not by accident.

This is not a deficiency to be solved by better models. A probabilistic caller is optimized for helpfulness over accuracy — its default response to ambiguity is to try harder, not to stop. The model that follows the access control instruction is the same model that follows the injected instruction. Prompt injection bypass rates range from 25% to 92% in controlled benchmarks. The TIP exploitation paper achieved remote code execution on every tested agent-LLM pair through crafted tool descriptions and return values.[^3] Prompt-as-ACL is worse than no strategy because it creates the illusion of control — the team believes access is governed when it is not.

Detection: "Does our system prompt contain access control instructions?" If yes, the D1 litmus test fails — publishing the system prompt reveals the full access control strategy and every restriction that can be bypassed. Fix: move the restriction to namespace governance — remove the tool from the namespace for callers who shouldn't have it, or bound its reach through execution-context scoping.

**Service account sprawl.**

*Diagnostic: D3 (Authorization-scope propagation).*

Across different transactions, system access is often accomplished via service accounts or fixed credentials not directly tied to the principal who initiated the transaction. Every tool runs under the same broad credential — and three tools with admin-level Salesforce credentials can reach everything. The namespace may be correctly scoped but the blast radius within the namespace is unbounded: the agent can only see three tools, but each tool can reach the entire backend. The deputy problem resurfaces at the execution-context level: the tool acts as a deputy carrying authority that belongs to no specific principal.

Detection: "Do our tools all run under the same service account?" If yes, D3 detects it: authorization-scope propagation is missing. Fix: execution-context scoping — each tool carries the correct authorization boundary for the transaction, not a shared default.

**Policy-as-architecture.**

*Diagnostic: D2 (Caller-sensitivity).*

Namespace boundaries are enforced by policies that are not backed by structural separation. The answer to "can the agent reach beyond its namespace?" is "no, the gateway blocks it" rather than "the namespace is the entirety of the agent's runtime." A discovery bypass, a server-side misconfiguration, or a prompt injection naming the tool directly can cross a policy boundary. The tools are outside the primary namespace but inside the effective namespace.

Detection: "Could the agent invoke a capability that exists somewhere in the infrastructure but isn't in its namespace?" If the boundary depends on a filter or policy rule applied *within* a shared runtime — where the filtered capabilities still exist and could be reached through a discovery bypass, misconfiguration, or direct invocation attempt — the boundary is policy-based. A gateway that routes different callers to entirely different backend instances is structural separation, not policy filtering — the distinction is whether the capabilities exist in the agent's runtime at all. Fix: structural separation — different runtime environments for different authorization contexts, where capabilities outside the namespace do not exist in the agent's reachable infrastructure. For standard-consequence operations where structural separation isn't warranted, policy-based boundaries with monitoring may be a reasonable tradeoff (§ Tradeoffs and Tensions).

**Trust leaks.**

*Diagnostic: D4 (Primary namespace).*

The trust level for a given operation is unpredictable. The agent might chain through a tool with broader credentials, invoke another agent with different access, use a non-tool capability to achieve something the primary tools don't allow, or traverse a shared service account that silently elevates the transaction. The primary namespace may be enumerated, but the authorization boundary flowing through it is not.

Detection: "For a given operation, can we predict exactly what authorization boundary it will execute under?" If the answer depends on which path the agent takes rather than on what the control plane enforces, trust is leaking — escaping the intended authorization boundary through unexpected paths. Fix: execution-context scoping at every boundary — each tool, each delegation, each internal service call carries the correct authorization for the transaction, enforced by infrastructure rather than by the agent's choice of path.

**Ghost tools.**

*Diagnostic: D5 (Effective namespace).*

Capabilities exist in the effective namespace that nobody authorized. A tool's internal service call that escalates to a broader API. A code execution environment the agent can invoke. An error response that leaks a connection string. A dynamically registered capability that appeared after deployment. These are ghost tools — they exist in the effective namespace but not in the primary namespace the team governs.

Detection: "Does our agent have access to capabilities beyond the tools we explicitly authorized?" This is the hardest to spot because you're looking for things you don't know are there. D5 detects it: the effective namespace is larger than the primary namespace. Fix: contracted execution boundaries (Axis 2, Level 3) — trace internal service calls, contract error responses (Factor VI), enumerate non-tool capabilities, close the gap between primary and effective namespace.

---

## Industry Examples

Three domains, each illustrating how industry-specific constraints shape Factor IV's application.

### Hospitality: structural separation with role-based authorization

A guest-facing agent connects to an MCP server exposing `search_bookings_on_behalf_of_guest` and `submit_service_request` — tools scoped to the authenticated guest's own records in Salesforce and the PMS. A staff-facing agent connects to a separate server exposing `search_bookings_on_behalf_of_staff`, `override_rate`, `comp_room`, and `view_all_guests` — tools carrying staff-level authorization that can operate across any guest's records, process rate adjustments through Stripe, and write to audit-sensitive CRM fields. Even operations that sound similar — searching bookings — are architecturally different tools with different names, different authorization contexts, and different execution-context scopes. The guest agent cannot escalate to staff operations because those operations do not exist in its namespace, and the operations it can reach are bounded by its execution context.

The response path illustrates Axis 2 at work. The guest's `search_bookings_on_behalf_of_guest` returns the guest's own booking details — dates, room type, folio balance. The staff's `search_bookings_on_behalf_of_staff` returns the same booking plus internal notes, override history, loyalty tier data, and payment method details. The execution context determines not just which records the tool can query but which fields appear in the response. The authorization boundary holds in both directions.

### Financial services: regulatory authorization in the execution context

A client-facing advisor agent on a wealth management platform sees portfolio viewing and trade-request tools scoped to the advisor's assigned client book. Each tool's execution context carries the advisor's regulatory profile — their Series 7 license scope, their firm's compliance rules, their client authorization list. A compliance agent sees audit and account-freeze tools scoped to the full book, with execution contexts carrying compliance-officer credentials and regulatory authority. Payment tools in either namespace carry the transaction-initiating principal's authorization — not a shared service account. Cedar policies evaluate each invocation against the principal's regulatory permissions. The advisor agent cannot freeze an account; the compliance agent cannot initiate a trade. The tools for each operation exist only in the appropriate namespace, and the execution context bounds what each tool can reach within that namespace.

The delegation case is particularly visible here. If a portfolio-management agent delegates a tax-lot optimization request to a specialized analytics agent, the analytics agent's namespace should include read-only tools for the specific client's positions — not the advisor's full client book. The effective authorization is the intersection of the principal's (the client), the advisor agent's (the assigned book), and the analytics agent's (read-only analytics). Without this intersection, delegation launders authorization: the analytics agent inherits the advisor's full book when it should only see one client's positions.

### Healthcare: minimum necessary as namespace governance

A patient-facing agent sees appointment scheduling and medication-refill-request tools scoped to the authenticated patient's own records. A clinician-facing agent sees the same patient's records plus clinical notes, lab ordering, and prescription tools — with the clinician's NPI, institutional credentials, and DEA registration in the execution context. The prescription tool exists only in the clinician's namespace; the refill-request tool (which triggers a clinician review, not a direct prescription) exists only in the patient's. Both namespaces touch the same underlying patient record through different authorization boundaries.

HIPAA's minimum necessary standard is the regulatory expression of the same principle: access should be the minimum necessary for the purpose. Factor IV's governance spectrum is how that standard becomes architectural. The patient agent's execution context filters what the patient sees — appointment dates and medication names, not clinical notes or lab values. The clinician's execution context reveals the clinical record scoped to the encounter — the patient's relevant history, not the full chart of every patient the clinician has ever seen. The response path is where minimum necessary is most often violated: a tool that queries correctly but returns the full record to the reasoning layer has failed the standard even if the namespace is perfectly scoped.

---

## Tradeoffs and Tensions

**Governance depth vs. consequence severity.** Per-call namespace materialization at the role level is the prescription. Taken to its extreme — per-operation, per-parameter namespace construction — it produces infrastructure burden that exceeds the security benefit for standard operations. The spectrum from Level 0 to Level 3 is not a checklist where anything below the maximum is a failure. Policy-based namespace generation (Level 2) with monitoring may be the right engineering decision for read-only informational capabilities or standard-consequence operations. Structural separation is warranted for mutation capabilities touching financial or health records. The important thing is that the team has made a deliberate decision about where on the spectrum each class of operation sits, rather than defaulting to Level 0 because no one evaluated the surface.

**Developer experience vs. operational discipline.** Structural separation means maintaining multiple runtime environments, multiple capability registrations, multiple deployment configurations. This is real operational overhead — more infrastructure to manage, more configuration to keep consistent, more deployment surface to monitor. The cost is worth paying. Enterprise agentic systems operate on behalf of different principals with different authorization boundaries, touching systems with regulatory and financial consequences. The alternative — a single namespace where every agent sees everything — is not a simpler architecture. It is an ungoverned one. The friction of maintaining structurally separated namespaces is the friction of doing access control correctly. The cost of not doing it is in the next section.

**Namespace scope vs. capability design.** A common objection to tight namespace scoping is that it limits the agent's usefulness — a guest-facing agent that can only see `search_bookings_on_behalf_of_guest` and `submit_service_request` can't suggest upgrading a room or extending a stay. But the objection assumes usefulness requires a wide namespace. It doesn't. It requires well-designed capabilities. A well-designed `manage_booking` tool that accepts intent ("I'd like to extend my stay") and routes to the appropriate backend operation through the control plane is more useful than five implementation-leaked tools that each expose one backend endpoint — and the namespace stays small. Factor III addresses how to design capabilities at that level of abstraction. The framework's position is that the reasoning layer should do what it is responsibly good at — reasoning about intent — and no more. A tightly scoped namespace is not a constraint on the agent's usefulness. It is the architectural expression of enabling an agent to maximize its helpfulness.

---

## The Cost of Unbounded Access

What happens when namespace governance is absent — when the agent's namespace is whatever happens to be connected and execution contexts are whatever was deployed.

**Prompt injection as privilege escalation.** When the namespace is ungoverned, a compromised customer-facing chatbot inherits the full namespace — including staff-level operations. Rehberger's Month of AI Bugs (August 2025) documented one vulnerability per day across ChatGPT, Cursor, Claude Code, Copilot, and others, with tool-invocation-based exploits as a recurring pattern.[^4] The 39C3 "AI Kill Chain" — prompt injection, confused-deputy behavior, automatic tool invocation — operates on the namespace the agent can see.

**The leaked system prompt as access control map.** When prompt-as-ACL is the strategy, a leaked system prompt reveals the full namespace and every access restriction that can be bypassed. The TIP exploitation paper demonstrated prompt extraction and subsequent exploitation on every tested agent-LLM pair.[^3]

**Tool descriptions as attack surface.** The Attractive Metadata Attack demonstrated that tool descriptions visible to the agent are themselves an attack surface — crafted metadata induces the agent to select attacker-controlled tools without any prompt injection of the user input.[^5] ToolHijacker automated this across multiple agent frameworks. The namespace's contents determine the attack surface; the namespace's size determines the blast radius.

**Credential exposure through the namespace.** API keys, service account identifiers, OAuth tokens, and connection strings embedded in capability configurations travel with the namespace. When the namespace is unbounded, every credential attached to every capability is exposed to the agent — and to anything that can extract the agent's context. A compromised agent with an ungoverned namespace doesn't just see capabilities it shouldn't have. It sees the credentials those capabilities carry: database connection strings, third-party API keys, service account tokens with broad permissions. The credential surface scales with the namespace. Bounding the namespace bounds the credential exposure.

---

## Lineage

Factor IV's intellectual ancestry runs through sixty years of capability-security theory. The departure — and the reason the control plane is privileged — is that LLM agents are not cooperative subjects who can be trusted with authority.

**The capability discipline.** Dennis and Van Horn's 1966 formalization of capability-based addressing established the foundational rule: if a subject does not hold a capability for an object, it has no way to name, address, or invoke that object. No ambient authority.[^6] Miller's 2006 dissertation developed this into the Principle of Least Authority and the object-capability model.[^7] Factor IV defines what this discipline looks like at the control plane layer. Confinement, delegation enforcement, attenuation, revocation — all four manifest as control plane operations. Confinement: the control plane materializes the per-call namespace and enforces the boundary. Delegation: when a downstream agent needs a subset of capabilities, the control plane materializes that agent's namespace scoped to the delegated subset. Attenuation: the execution-context scoping pattern encodes the upstream caller's authorization boundary into the tool's execution context, bounding reach within the namespace. Revocation: dynamic namespace reconstruction when authorization state changes. What the capability discipline looks like at the reasoning layer — how an agent might itself reason about authority — is a different project's concern, not ours.

**Least privilege relocated.** Saltzer and Schroeder's 1975 principle — "every program and every user should operate using the least set of privileges necessary to complete the job" — presupposes a cooperative subject.[^8] NIST SP 800-207 extended it to untrusted networks with per-session evaluation.[^9] Factor IV relocates the subject entirely: the unit of privilege is not the agent but the per-call composed namespace, scoped to the caller's role.

**The confused deputy dissolved.** Hardy's 1988 diagnosis (§ How This Works) has been the field's organizing problem for decades.[^10] Every tradition that followed — capabilities, POLA, zero trust — addressed it by making the deputy more trustworthy. Factor IV's departure is that a probabilistic caller cannot be a deputy at all. A deputy holds authority and exercises it on behalf of a caller — that role requires the ability to distinguish whose authority is in play and to exercise it faithfully. A probabilistic caller has neither ability, not as a deficiency in current models but as a property of the architecture. The response is not to build a better deputy. It is to eliminate the deputy role entirely: the agent expresses intent, the control plane holds authority, and the conditions for confusion do not arise.

**Convergent recognition.** Multiple independent communities arrived at the same diagnosis in 2024–2026. MiniScope formalized least-privilege tool authorization as a security game.[^2] AWS Bedrock AgentCore operationalized Cedar-based policy evaluation at the gateway, with explicit "Confused Deputy prevention" framing. Willison's lethal trifecta named the conditions under which agent compromise becomes exfiltration: private data, untrusted content, and external communication in the same namespace.[^11] Rehberger's Month of AI Bugs provided the empirical floor — one tool-invocation vulnerability per day across the major agent platforms.[^4] Factor IV's contribution within this convergence is the litmus test, the governance spectrum, and the no-deputies principle that gives practitioners a design criterion rather than a vulnerability taxonomy.

---

## Related Factors

This factor addressed what exists in the caller's namespace and how authorization boundaries are enforced within it. For how the control plane itself is architected — the layer that owns namespace materialization, credential management, and governance at every consequence level — see Factor I: Governed Operations. For how the capabilities in the namespace should be designed to express intent rather than expose implementation — see Factor III: Intent-Based Communication. For how state mutations that pass through the namespace are mediated by the control plane — see Factor II: Deterministic Mutations. For what happens when the reasoning layer retries an operation within a bounded namespace — see Factor V: Safe Retries. For how the control plane communicates outcomes back to the reasoning layer when operations fail — see Factor VI: Recovery Contracts. For how every operation that passes through the namespace is recorded — see Factor VII: Structural Observability.

---

## Notes

[^1]: Bhattarai, B. and Vu, T. "Trustworthy Agentic AI Requires Deterministic Architectural Boundaries." arXiv:2602.09947, Los Alamos National Laboratory, February 2026. The impossibility argument — no training-only procedure can guarantee deterministic command-data separation in autoregressive transformers — supports the "no deputies" principle: better training does not produce a trustworthy deputy.

[^2]: Zhu, J., et al. (2025). MiniScope: A least privilege framework for authorizing tool calling agents. arXiv:2512.11147. Formalizes least-privilege tool authorization as a security game; reconstructs permission hierarchies from existing OAuth scopes.

[^3]: Debenedetti, E. et al. (2024). AgentDojo: A Dynamic Environment to Evaluate Attacks and Defenses for LLM Agents. The TIP exploitation paper achieved remote code execution on every tested agent-LLM pair through crafted tool descriptions and return values. Prompt injection bypass rates: 25–92% across benchmarks.

[^4]: Rehberger, J. (2025). Month of AI Bugs [vulnerability disclosure series, August 2025]. Embrace The Red. One tool-invocation vulnerability per day across ChatGPT, Cursor, Claude Code, Copilot, and others. See also: "Agentic ProbLLMs: Exploiting AI Computer-Use and Coding Agents," 39th Chaos Communication Congress, December 2025.

[^5]: Wang, X. et al. (2025). Attractive Metadata: Manipulating AI Agent Tool Selection via Metadata Injection. arXiv:2508.02110. Tool descriptions visible to the agent are themselves an attack surface — crafted metadata induces tool selection without prompt injection of user input.

[^6]: Dennis, J. B., & Van Horn, E. C. (1966). Programming semantics for multiprogrammed computations. *Communications of the ACM*, 9(3), 143–155.

[^7]: Miller, M. S. (2006). *Robust Composition: Towards a Unified Approach to Access Control and Concurrency Control* [Doctoral dissertation, Johns Hopkins University].

[^8]: Saltzer, J. H., & Schroeder, M. D. (1975). The protection of information in computer systems. *Proceedings of the IEEE*, 63(9), 1278–1308.

[^9]: National Institute of Standards and Technology. (2020). *Zero Trust Architecture* (NIST SP 800-207).

[^10]: Hardy, N. (1988). The confused deputy (or why capabilities might have been invented). *ACM SIGOPS Operating Systems Review*, 22(4), 36–38.

[^11]: Willison, S. (2025, June 16). The lethal trifecta for AI agents: Private data, untrusted content, and external communication. Simon Willison's Weblog.
