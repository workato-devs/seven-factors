# Extended Guidance Archetype — Best-of-Breed Skeleton

**What this is.** A universal section-by-section skeleton for extended guidance documents that accompany a principle-level statement. Derived from structural analysis of eleven durable "principle + guidance" layered documents — including AWS Well-Architected Framework, Fowler's *Patterns of Enterprise Application Architecture*, the Gang of Four *Design Patterns*, Google's *Site Reliability Engineering*, and the NIST Cybersecurity Framework. Each section earns its place by appearing in at least three of these comparables or by addressing a gap the analysis identified.

**Why this exists.** The principle says *what* to believe. The extended guidance layer equips the practitioner to *act on* that belief — to assess their current state, recognize failure, implement the principle, understand the tradeoffs, and know when the guidance doesn't apply.

**Scale target.** Fowler pattern entry density (~8 pages / ~3,500–5,000 words). Not AWS pillar whitepaper density (60,000+ words). Deep enough to be a reference; brief enough to read in a sitting.

---

## The Skeleton

### § 0 — Desired Outcome
*1–3 sentences. What does "good" look like when this factor is fully implemented?*

> **Rationale.** AWS WAF post-2022 leads every best practice with a "Desired outcome" statement. NIST CSF subcategories are written as outcome statements. This orients the reader before any diagnostic or mechanism detail. The reader knows what they're aiming for before they learn how to assess whether they're there.
>
> **Comparable precedent.** AWS WAF (Desired Outcome, per best practice), NIST CSF (Subcategory outcome statements), Azure WAF (implicit in Design Principles).
>
> **What it is NOT.** This is not the principle's bolded assertion restated. The principle says "this is the factor." The desired outcome says "this is what your system looks like when it embodies the factor." Concrete, observable, testable.

---

### § 1 — Terms of Art
*Define diagnostic language the field lacks. Precise, enumerable criteria.*

> **Rationale.** The GoF introduces pattern names that become shared professional language. Fowler's PoEAA defines terms (Active Record, Data Mapper, Unit of Work) that practitioners use decades later. When a factor introduces vocabulary the industry doesn't have, defining it early gives the reader a lens for everything that follows.
>
> **Comparable precedent.** GoF (pattern name as term of art), Fowler PoEAA (named patterns become industry vocabulary), AWS WAF (defined best practice area names), NIST CSF (Function/Category/Subcategory taxonomy).
>
> **Conditional.** Only when the factor introduces terms the field needs. If the factor uses only established vocabulary, this section is omitted and the skeleton begins at § 0 + § 2.

---

### § 2 — How This Works
*The positive mechanism. How the principle operates in practice — the architectural pattern, the structural approach, the "what you build."*

> **Rationale.** Fowler's "How It Works" is the paradigmatic version — every PoEAA pattern has one. The GoF has "Implementation" + "Participants" + "Collaborations." AWS WAF has "Implementation guidance" per best practice. Google SRE's Practices chapters are extended "how it works" treatments. A reader who can diagnose problems and recognize anti-patterns but cannot implement the solution has been given a compass without a map.
>
> **Comparable precedent.** Fowler PoEAA (How It Works — named section), GoF (Implementation + Participants + Collaborations), AWS WAF (Implementation guidance), Google SRE (Part III Practices), EIP (Extended Solution Description).
>
> **Principle-mechanism discipline.** This section explains the mechanism as *one proven approach* — not THE approach. Language like "one structural pattern that satisfies this principle" or "a common approach" marks the mechanism as exemplar, not mandate. The mechanism is replaceable cladding; the principle (stated in the principle layer and the Desired Outcome) is the structure. This is the Approach B+ from the durability research.
>
> **Why this comes before diagnostics.** Fowler and the GoF both place mechanism before assessment. The reasoning: you need to understand what the healthy pattern looks like before you can evaluate deviations from it. A diagnostic framework without a prior "this is what healthy looks like" section forces the reader to infer the positive case from the negative cases (anti-patterns). That inference is unreliable. Show the target, then show how to measure distance from it.

---

### § 3 — When This Applies (and When It Doesn't)
*Applicability boundaries. Contexts where the guidance applies strongly, contexts where it applies partially, and contexts where alternative approaches are more appropriate.*

> **Rationale.** Fowler calls this *the* most important structural decision in pattern writing: "One of the most useful things I do when understanding a pattern is ask, 'When would I not use this pattern?'" His PoEAA entries enforce this with a dedicated "When to Use It" section. The GoF has a named "Applicability" section. Documents that omit this produce guidance that readers either over-apply or dismiss.
>
> **Comparable precedent.** Fowler PoEAA (When to Use It — named section), GoF (Applicability — named section), OWASP (Description lists vulnerable conditions, implicitly scoping applicability).
>
> **What this covers.** Applicability varies across: synchronous vs. asynchronous agent patterns, single-tool vs. multi-tool namespaces, human-facing vs. agent-to-agent callers, greenfield vs. brownfield adoption, and scale (hobby project vs. enterprise). This section names those axes and says where the factor's guidance is strongest, weakest, and inapplicable.

---

### § 4 — Diagnostic Framework
*Dimensions of evaluation. How to assess whether your system embodies this factor. Specific, answerable questions.*

> **Rationale.** AWS WAF's 57 open-ended review questions are the most developed example — they converted passive reading into active assessment and powered 10+ years of architectural reviews. Azure WAF adds a Design Review Checklist. NIST CSF uses Organizational Profiles. Without a diagnostic mechanism, guidance becomes reference material that teams browse rather than a framework they apply.
>
> **Comparable precedent.** AWS WAF (57 review questions, coded by best practice area), Azure WAF (Design Review Checklist with Advisor scoring), NIST CSF (Organizational Profiles: current state vs. target state), Google SRE Workbook (assessment companion to principles book).
>
> **Structural note.** Diagnostic questions should be open-ended ("How do you ensure that...?"), not yes/no. AWS WAF learned this — open-ended questions produce conversation; yes/no questions produce checkboxes. Each diagnostic dimension should map to a specific aspect of the factor's principle. If the factor has four dimensions of evaluation, there should be four diagnostic questions (or question clusters).
>
> **Severity/priority markers.** Diagnostic dimensions should carry priority indicators — which dimensions matter most for teams just starting, which matter for mature implementations.

---

### § 5 — Named Anti-Patterns
*Failure modes named at the same granularity as the positive guidance. One per diagnostic dimension where possible.*

> **Rationale.** AWS WAF post-2022 added 3–8 named anti-behaviors per best practice. GoF lists both benefits and liabilities in Consequences. OWASP's Description section functions as failure-mode documentation. Raymond uses cultural counter-examples. Guidance without named failure modes produces compliance-oriented reading ("did we check the box?") rather than judgment-oriented reading ("are we avoiding the traps?").
>
> **Comparable precedent.** AWS WAF (Common anti-patterns per best practice, 3–8 named), GoF (Consequences: liabilities), OWASP (vulnerability condition descriptions), Raymond (cultural counter-examples in prose).
>
> **Structural requirement.** Each anti-pattern should include: (1) a name, (2) a description of the failure, (3) a diagnostic test ("how do you recognize this?"), and (4) a structural fix ("what do you do about it?"). The diagnostic test should map back to a specific dimension from § 4. The fix should map back to the mechanism described in § 2.

---

### § 6 — Industry Examples
*Concrete examples across multiple domains. Show the principle operating in different contexts to prove generality.*

> **Rationale.** GoF requires a minimum of three "Known Uses" per pattern from real systems. EIP implements each pattern across JMS, MSMQ, TIBCO, and BizTalk. Fowler's examples use Java and C# but are designed to be "skippable — the pattern should be understandable without the code." Google SRE uses named production incidents. The common thread: examples ground the principle; the principle outlives the examples.
>
> **Comparable precedent.** GoF (Known Uses ≥ 3 real systems), EIP (multi-platform implementations), Fowler PoEAA (Java/C# code, skippable), Google SRE (named incident narratives), OWASP (numbered attack scenarios with code).
>
> **Cross-domain requirement.** At minimum two industries beyond the framework's primary exemplar. Examples should use named vendors and real system types (not "System A" and "System B") — this is what makes them falsifiable. But technology-specific implementation details should be structured so they're skippable without losing the conceptual argument (Fowler's skippability principle).

---

### § 7 — Tradeoffs and Tensions
*What you sacrifice or risk by following this guidance. Tensions with other factors. Competing concerns.*

> **Rationale.** GoF's "Consequences" section lists benefits AND liabilities. Azure WAF uses dedicated tradeoff icons and documentation. Fowler's "When to Use It" discusses alternatives. Documents that present guidance as unambiguously good — without naming what you sacrifice — feel naive to experienced practitioners.
>
> **Comparable precedent.** GoF (Consequences: benefits and liabilities for every pattern), Azure WAF (dedicated tradeoff documentation per pillar), Fowler PoEAA (alternatives discussed in When to Use It), AWS WAF (cross-pillar tension notes in main framework document).
>
> **What this covers.** Inter-factor tensions (following this factor may create pressure on another factor), adoption cost (the engineering effort to implement the principle), organizational tensions (the factor may conflict with existing team structures, vendor contracts, or legacy systems), and edge cases where strict adherence produces worse outcomes than pragmatic compromise.
>
> **Why this is separate from "The Cost of [X]."** "The Cost of [X]" argues the cost of *non-compliance*. This section argues the cost of *compliance* — a fundamentally different rhetorical move. The first says "here's what you lose if you don't follow the principle." The second says "here's what you pay if you do — and here's why it's still worth it." Both are needed. Experienced practitioners are suspicious of guidance that only argues one side.

---

### § 8 — The Cost of [Factor-Specific Noun]
*Quantified or quantifiable stakes. What happens operationally when this factor is violated.*

> **Rationale.** This section has no direct analogue in the classic pattern literature (Fowler, GoF), but appears strongly in operational frameworks. AWS WAF assigns High/Medium/Low risk per best practice. OWASP ranks by severity using quantitative factors. Google SRE's entire error budget concept is a cost-of-violation framework. The section earns its place because the framework's audience includes decision-makers who need operational metrics, not just architectural elegance.
>
> **Comparable precedent.** AWS WAF (risk levels per best practice), OWASP (quantitative severity ranking), Google SRE (error budgets as cost-of-violation framework), NIST CSF (Tiers as implicit cost gradient).
>
> **What this covers.** Operational metrics: error rates, context window consumption, decision surface area, compensation event frequency, incident recovery time. The argument should compound — showing how violations in one dimension cascade into costs across multiple dimensions.

---

### § 9 — Lineage
*Credit the architectural tradition. Name the intellectual predecessors. Connect to the history of the discipline.*

> **Rationale.** Raymond's UNIX rule explanations are the paradigmatic example — he draws authority from named pioneers (McIlroy, Thompson, Pike, Spencer), using blockquoted elder wisdom to anchor each rule in a tradition. The Reactive ecosystem evolved explicitly across versions. NIST CSF documents its version history. The structural lesson: guidance that connects to a tradition earns more trust than guidance that appears to invent itself.
>
> **Comparable precedent.** Raymond (pioneer quotes, UNIX tradition throughout), Reactive ecosystem (Manifesto v1 → v2 evolution documented), NIST CSF (version comparison 1.0 → 2.0), Hoffman (quotes original factor, temporal pivot to "what changed").
>
> **What this is NOT.** This is not an academic literature review. It is a practitioner-facing acknowledgment that the principle has a pedigree — that it was not invented by this framework but applied to a new boundary (the probabilistic caller). The closing move should be: "the principle is established; the context is new."

---

### § 10 — Related Factors
*Navigational breadcrumbs. Cross-references to adjacent factors and external resources.*

> **Rationale.** EIP patterns end with explicit "Related Patterns" lists. GoF has the same. AWS WAF links each best practice to related practices across pillars. NIST CSF maps subcategories to 50+ external standards. Real architectural decisions involve multiple principles simultaneously — a document that only reads linearly cannot model the interaction between concerns.
>
> **Comparable precedent.** EIP (Related Patterns — named section), GoF (Related Patterns — named section), AWS WAF (Related best practices across pillars), NIST CSF (Informative References to 50+ external standards).
>
> **Format.** ~60 words. Topical breadcrumbs, no argumentation, no boundary claims. Consistent format across all factors: "This factor addressed [X]. For [adjacent concern A], see Factor [N]: [Name]. For [adjacent concern B], see Factor [N]: [Name]."

---

## Section ordering rationale

The ordering follows a **practitioner journey**: orient → understand → scope → assess → recognize failure → see proof → weigh costs → understand consequences → ground in tradition → navigate outward.

| Position | Section | Journey stage | Why this position |
|---|---|---|---|
| § 0 | Desired Outcome | **Orient** | Before anything else, the reader knows what they're aiming for. |
| § 1 | Terms of Art | **Equip** | Give the reader the vocabulary they'll need for everything that follows. Conditional — only if the factor introduces new terms. |
| § 2 | How This Works | **Understand** | Show the positive mechanism before assessment. You need to know what healthy looks like before you can measure deviation. (Fowler, GoF ordering.) |
| § 3 | When This Applies | **Scope** | Before diagnostics, the reader knows whether this guidance applies to their context. Prevents over-application and premature dismissal. (Fowler's "most important structural decision.") |
| § 4 | Diagnostic Framework | **Assess** | Now the reader can evaluate their own system. Diagnostic questions are actionable because the reader understands the mechanism (§ 2) and knows the guidance applies to them (§ 3). |
| § 5 | Named Anti-Patterns | **Recognize failure** | Anti-patterns are most useful after the reader has a diagnostic framework — they become "this is what a failing diagnostic looks like in practice." |
| § 6 | Industry Examples | **See proof** | Generality proof. The reader has the framework; now they see it applied across domains. |
| § 7 | Tradeoffs and Tensions | **Weigh costs of compliance** | After seeing the full positive case (§ 0–6), the reader gets the honest counterargument: what this costs and where it creates tension. This is where experienced practitioners decide whether to trust the document. |
| § 8 | The Cost of [X] | **Weigh costs of non-compliance** | The counterbalance to § 7. After seeing what compliance costs, the reader sees what non-compliance costs. The two sections together let the practitioner make an informed decision. |
| § 9 | Lineage | **Ground in tradition** | Intellectual close. The principle has a pedigree. The reader trusts the guidance more knowing it extends a 50-year tradition rather than inventing itself. |
| § 10 | Related Factors | **Navigate outward** | Navigational close. The reader knows where to go next. |

