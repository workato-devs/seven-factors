# Contributing to the Seven Factors

## What this framework is and isn't

The Seven Factors is a principle-plus-guidance framework — not a specification, not a product, and not a certification standard. It introduces vocabulary for architectural concepts the industry currently lacks, names the failure modes teams encounter in production, and provides diagnostic tools for evaluating existing systems. Contributions should serve practitioners: the architect assessing a system, the engineer reviewing a pull request, the technical leader deciding whether to ship.

The intended reader is a practitioner with production experience who has encountered the problems these factors describe. Write for that person.

---

## Structure

Each factor has two layers. Both have specific editorial roles.

**The principle** (`principle.md`) states the factor's core assertion. It is short (~400–600 words), assertion-dense, and designed to stand alone. It does not instruct — it describes what a healthy implementation looks like ("the control plane does X") rather than telling the reader what to do ("you should do X"). It does not hedge. It closes with a specific failure mode and its consequence. Changes to principle files require strong justification — these are the stable surface of the framework.

**The extended guidance layer** (`guidance.md`) equips the practitioner to act on the principle. It follows the structure in [contributing/archetype.md](contributing/archetype.md). It covers: desired outcome, terms of art (where the factor introduces framework vocabulary), mechanism, applicability, diagnostic dimensions, named anti-patterns, industry examples, tradeoffs, the cost of ignoring the factor, lineage, and cross-factor navigation. Guidance files are living documents and accept substantive contributions.

---

## Editorial standards

**Vocabulary.** Six terms of art define the framework's core concepts — documented in [contributing/terms-of-art.md](contributing/terms-of-art.md). Use these consistently.

| Term | Owns | Replaces |
|---|---|---|
| Reasoning layer | Factor I | "agent," "LLM," "orchestrator" when used to name the intent-owning component |
| Control plane | Factor I | "middleware," "API gateway," "ESB" when used to name the consequence-owning layer |
| Probabilistic caller | Factor I/VI | "AI agent," "LLM" used as caller-type labels |
| Deterministic caller | Factor I | "traditional system," "coded caller" |
| Intent-aligned | Factor III | "good API design" when applied to the probabilistic-caller boundary |
| Implementation-leaked | Factor III | "bad API design," "information hiding violation" at this boundary |

Use domain-neutral vocabulary. "Contextual identifier" not "business identifier." "Domain practitioner" not "business user." "Namespace" not "tool inventory." The framework applies to SRE agents, coding agents, and clinical agents — not only to business-domain operations.

**Caller framing.** Operations are caller-agnostic: "a human asking a chatbot, an agent delegating via A2A, a scheduled job" — this framing appears in every factor. Do not narrow to one caller type.

**Probabilistic caller behavioral characterization.** Use Factor VI's canonical language when characterizing how probabilistic callers respond to ambiguity: "optimized for helpfulness over accuracy, which means its default response to ambiguity is to try harder, not to stop."

**Response-path coverage.** Every factor's diagnostic framework should address what comes back, not just what goes in. A tool that correctly scopes its query but returns unbounded results has a response-path failure.

**Economy.** Factor V V.2 is the benchmark for prose economy. Every sentence should advance its section's purpose. Cut: self-congratulatory framing, chronological sidetracks, editorial commentary, defensive hedging, restatements of the preceding paragraph.

**Cross-factor references.** Use factor names: "Factor III: Intent-Based Communication" or simply "Factor III." Link to the appropriate principle or guidance file. When characterizing a neighboring factor's territory, use its canonical framing — Factor III owns "interface composition," not "namespace design" (that is Factor IV).

---

## What belongs in a pull request

**Changes to principle files:** Require a clear rationale tied to the framework's editorial standards or vocabulary decisions. Changes to the bolded assertions require especially strong justification — these are the most-quoted and most-stable sentences in each factor.

**Changes to guidance files:** Add a section, extend a diagnostic dimension, add an industry example, fix a cross-factor reference, apply updated vocabulary. Guidance files should follow the archetype structure — if you are adding a new major section, confirm it has a precedent in the archetype.

**New footnotes and citations:** The framework grounds its claims in production evidence and peer-reviewed research. New empirical claims should be footnoted. Practitioner reports (named if possible) are valid evidence.

---

## What does not belong

- Prescriptive schemas or specific implementation libraries — the framework names what the architecture must do, not how to implement it in any particular stack.
- Product endorsements — vendor examples are used to illustrate principles, not to recommend specific tools.
- Principle changes that soften assertions — the principle layer is designed to be argued about, not hedged.

---

## Reference documents

- [contributing/archetype.md](contributing/archetype.md) — section-by-section skeleton for extended guidance documents
- [contributing/terms-of-art.md](contributing/terms-of-art.md) — the six qualifying terms of art and their definitions
