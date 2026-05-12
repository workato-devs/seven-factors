# The Seven Factors of the Agentic Control Plane — Introduction

## Introduction

In the current era of enterprise software, organizations connect AI agents to systems of record — CRMs, payment processors, ERPs, identity providers, clinical systems, financial platforms. But connecting an agent to a system of record is not the same as governing what it does there. The protocols are deliberately thin. The enterprise concerns they leave empty — authentication, authorization, transaction integrity, compliance — are where production deployments break or stall. The seven factors of the agentic control plane is a methodology for building a deterministic control plane beneath these agents: the layer that must be correct regardless of which LLM, framework, or agent architecture reasons above it.

The seven factors build from a key architectural distinction: every agentic application has two layers. The **reasoning layer** is the component that owns intent — it interprets requests, selects operations, and decides what should happen. It may be a single tool-calling LLM, a multi-step workflow engine, or a fully autonomous multi-agent system. The **control plane** is everything beneath it: the deterministic machinery that authenticates to enterprise systems, enforces authorization policies, validates business rules, sequences multi-system transactions, and produces the audit trail. The reasoning layer owns intent. The control plane owns consequences.

These are not equal partners. The reasoning layer is probabilistic — it will sometimes misinterpret, retry unnecessarily, or hallucinate a recovery strategy. The control plane must be correct anyway. It enforces invariants the reasoning layer cannot be trusted to maintain: a payment is charged exactly once, a patient record is modified only by an authorized clinician, a trade is executed within compliance boundaries no matter what the agent believes it was asked to do.

The seven factors describe what a healthy control plane looks like. The control plane's responsibilities do not change with the agent's autonomy level — they intensify.

## Background

Existing frameworks — HumanLayer's 12-Factor Agents, Google's agent design patterns — address the agent side of the boundary: how to build better reasoning, how to structure prompts, how to orchestrate multi-step plans. We address the other side: the enterprise infrastructure that must be correct regardless of what sits above it.

The contributors to this document have built and operated enterprise integration and automation infrastructure at scale — over one trillion workflow executions across thousands of enterprise customers. This includes production agentic deployments where AI agents perform real operations against systems of record through MCP servers, A2A protocols, and custom tool interfaces. This document synthesizes what we have observed about what breaks and what holds when probabilistic reasoning meets deterministic enterprise systems.

Our motivation is to provide a shared vocabulary for the architectural concerns that every team building production agentic applications will encounter, and to offer principles that are immediately actionable. The format is inspired by Adam Wiggins' Twelve-Factor App.

## Who should read this document?

- **Engineers building agents that operate against enterprise systems.** Your frameworks optimize the reasoning layer — intent, orchestration, tool selection — though they don't use that term. Other approaches to enterprise safety try to constrain the reasoning layer, fighting its probabilistic nature to fit enterprise context. The Seven Factors take the opposite position: build the deterministic control plane beneath it, and correctness guarantees become architectural rather than behavioral. This is how you maximize reasoning layer strength and enterprise safety at the same time.

- **Engineers building the enterprise platforms those agents call into.** Your systems already enforce authorization, business rules, and transaction integrity for deterministic callers. Probabilistic callers change that contract. These factors describe the architecture for the boundary layer your systems now need.

- **Security and compliance teams** evaluating agentic deployments.

- **Technical leaders** deciding how much trust to place in AI agents operating against systems of record.

## The Factors

1. [I. Governed Operations](factors/i-governed-operations/principle.md) — *No enterprise concern delegates to an agentic protocol*
2. [II. Deterministic Mutations](factors/ii-deterministic-mutations/principle.md) — *All state mutations belong to the control plane*
3. [III. Intent-Based Communication](factors/iii-intent-based-communication/principle.md) — *Tool boundaries follow intent, not implementation*
4. [IV. Bounded Access](factors/iv-bounded-access/principle.md) — *Each caller sees only the capabilities its role requires*
5. [V. Safe Retries](factors/v-safe-retries/principle.md) — *Every mutation is safely retried by a probabilistic caller*
6. [VI. Recovery Contracts](factors/vi-recovery-contracts/principle.md) — *The reasoning layer never guesses at state*
7. [VII. Structural Observability](factors/vii-structural-observability/principle.md) — *Every agent action is reconstructable by architecture, not investigation*
