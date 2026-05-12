# Terms of Art — Seven Factors of the Agentic Control Plane

**What this is.** The framework's qualifying terms of art — six terms that pass a four-prong test — and the qualification test itself, for contributors proposing new terms.

**Why this exists.** The framework says "here's how to think about the architecture." It doesn't coin vocabulary for every mechanism, failure mode, and design position within that architecture. Six terms qualify as terms of art. Everything else in the framework is a consequence practitioners can derive from those six, a design recommendation that follows from them, or a named anti-pattern that violates them.

---

## 1. The Qualification Test

A term of art is something a practitioner would use *outside* the document — in a code review, in an architecture discussion, in a post-mortem. It enters the reader's working vocabulary and stays there. Four conditions, all required:

**Prong 1: It names something the practitioner currently cannot name.** Before reading the paper, there was no concise way to point at this thing. The industry lacks the word. If an existing term already covers the concept adequately — even imprecisely — this prong fails.

**Prong 2: It survives extraction from the document.** If you pulled the term out and dropped it into a glossary with no surrounding argument, it would still be useful. The term carries its own weight. If it only makes sense in the flow of the paper's argument, it's a rhetorical device, not a term of art.

**Prong 3: It changes what the reader sees.** After learning the term, the reader notices things they didn't notice before. The term is a diagnostic lens — it makes a previously invisible phenomenon visible and nameable in the field.

**Prong 4: It credibly and materially improves the industry to have this term.** The industry gets *better* because this term exists. Practitioners make better decisions, teams have more productive conversations, code reviews catch things they previously missed. This is the sufficiency test — a term can pass prongs 1–3 and still not be worth elevating if the industry benefit is marginal.

---

## 2. The Qualifying Set

Six terms pass the four-prong test. These are the framework's terms of art.

| # | Term | Owning factor | What it names |
|---|---|---|---|
| 1 | **Reasoning layer** | Factor I | The component that owns intent — and nothing beyond intent |
| 2 | **Control plane** | Factor I | The consequence-owning layer between probabilistic callers and systems of record |
| 3 | **Probabilistic caller** | Factor I (introduced); Factor VI (behavioral characterization) | The caller type that breaks traditional architecture assumptions |
| 4 | **Deterministic caller** | Factor I | The caller type traditional architecture assumed was universal — the baseline that broke |
| 5 | **Intent-aligned** | Factor III | The healthy interface pattern at the probabilistic caller boundary |
| 6 | **Implementation-leaked** | Factor III | The violation — implementation details visible to a probabilistic caller |

The six terms form a coherent structure. Terms 1–2 name the *architecture* (the two-layer model). Terms 3–4 name the *paradigm shift* (the caller type that necessitates the model, and the baseline it replaces). Terms 5–6 name the *interface diagnostic* (how to evaluate whether the boundary between the layers is correctly drawn). Everything else in the framework is a consequence of these six concepts, a design recommendation that follows from them, or a named anti-pattern that violates them.

### Supporting concepts — by category

These are well-named, editorially important, and should remain in the extended guidance. They are not terms of art. Categorizing them clarifies what editorial role each plays.

**Derived consequences** (things practitioners can reason to from the six terms of art):
Ownership inversion, audit record vs. developer trace, the confused retry.

**Named anti-patterns** (violations and failure modes — diagnostic tools, not standing vocabulary):
Prompt-as-ACL, context-loss re-initiation, misinterpreted success/failure, multi-agent duplicate initiation, the saga-initiation problem.

**Design recommendations** (positions the framework advocates, not vocabulary it coins):
Recovery contract, the consumer assumption, the three-way outcome.

**Implementation-level concepts** (mechanisms that matter to control plane builders):
Implementation-aligned, confinement principle, namespace materialization, execution-context scoping.

**Existing vocabulary applied** (industry terms the framework sharpens for the agentic context):
Observability telemetry, audit boundary.
