# The Seven Factors of the Agentic Control Plane

Enterprise software teams are connecting AI agents to systems of record — CRMs, payment processors, ERPs, clinical systems. The connections work. The governance doesn't. Agents act without authorization. Mutations happen without audit trails. Failures produce errors the reasoning layer cannot interpret. Recovery logic gets invented on the fly — and billed by the token.

The **Seven Factors** is a methodology for building the control plane that sits between AI agents and enterprise systems: the deterministic layer that must be correct regardless of which model, framework, or agent architecture reasons above it. The reasoning layer owns intent. The control plane owns consequences. It is modeled on Adam Wiggins' Twelve-Factor App and synthesizes what the contributors have observed across over one trillion workflow executions at enterprise scale.

The problems these factors address are emerging with agentic architectures. The solutions draw from decades of enterprise integration, distributed systems, and security engineering — applied to the novel boundary between probabilistic reasoning and deterministic systems.

---

## The Factors

| # | Factor | Principle |
|---|---|---|
| [I](factors/i-governed-operations/principle.md) | **Governed Operations** | *No enterprise concern delegates to an agentic protocol* |
| [II](factors/ii-deterministic-mutations/principle.md) | **Deterministic Mutations** | *All state mutations belong to the control plane* |
| [III](factors/iii-intent-based-communication/principle.md) | **Intent-Based Communication** | *Tool boundaries follow intent, not implementation* |
| [IV](factors/iv-bounded-access/principle.md) | **Bounded Access** | *Each caller sees only the capabilities its role requires* |
| [V](factors/v-safe-retries/principle.md) | **Safe Retries** | *Every mutation is safely retried by a probabilistic caller* |
| [VI](factors/vi-recovery-contracts/principle.md) | **Recovery Contracts** | *The reasoning layer never guesses at state* |
| [VII](factors/vii-structural-observability/principle.md) | **Structural Observability** | *Every agent action is reconstructable by architecture, not investigation* |

---

## How to Read This

Each factor has two layers.

**The principle** — a short, assertion-dense statement of the factor's core claim. Read these in sequence. Together they describe a complete architecture. Each is designed to be quoted, posted, and argued about.

**The extended guidance** — how to act on the principle. Diagnostic questions, named anti-patterns, worked examples, tradeoffs. Read these when you are assessing a system, designing a capability, or reviewing a pull request.

```
factors/
├── i-governed-operations/
│   ├── principle.md        # The principle (~500 words)
│   └── guidance.md         # How to act on it (~5,000 words)
├── ii-deterministic-mutations/
│   ├── principle.md
│   └── guidance.md
...
```

Start with [the introduction](introduction.md) if you are new to the framework. Start with [Factor I](factors/i-governed-operations/principle.md) if you want to understand the foundational architecture. Start with the factor closest to the problem you are debugging if you are in the middle of an incident.

---

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) and [contributing/archetype.md](contributing/archetype.md) for editorial standards and the extended guidance document structure.
