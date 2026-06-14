# Lucid — Design Devlog

This document traces the thinking behind [Lucid](link), a code clarity tool for understanding and modifying unfamiliar codebases. It's less about what I built and more about what I got wrong, why, and what I changed.

---

## Where it started

Two problems I kept running into:

**I couldn't read my own code.** State scattered everywhere, functions calling functions with no clear ownership. Every fix required reconstructing the entire dependency chain from scratch. Frontend code was the worst offender, but the problem showed up everywhere.

**I wanted to turn a mindmap into code and actually understand the result.** Not just generate something — I wanted output I could read, modify, and trust.

These felt like two separate problems. They turned out to share the same root: I couldn't see the structure of my own code.

---

## v0.1 — Local model workflow

**Constraint:** I didn't want to pay for API calls or live inside a chat window.

Started with a plan → code pipeline using local models. One model plans, another generates.

**What broke:** Small models produce vague plans and ignore format constraints. Got into planning loops — the model would replan indefinitely without generating anything. Restructured it as multiple choice at each step. Still unstable. This was language-agnostic — the instability showed up regardless of what kind of code I was generating.

Two observations pushed me toward a new hypothesis:

- Watching a model output code token by token felt wrong. Code isn't prose — a function signature is a pattern, not a prediction. The right answer for a standard function should be looked up, not generated.
- Most of the output had everything typed as `optional`. My interpretation: without type context, the model was hedging to pass basic checks. Technically valid, not useful.

**New hypothesis:** Code doesn't have that much entropy, especially for standard patterns. A search algorithm plus a template library should handle most cases — which is roughly how engineers approached this before LLMs.

Built `codegenerator` with SEQ/PAR/ROUTER plan logic and memory-based template references. Memory templates improved success rate, but consumed context window. Ran it overnight — out of memory. Ran it again — out of memory again. Disabled memory.

**What I learned:** Memory isn't the answer, not for my case at least. The model needs structure, not more context. The template library hypothesis was worth pursuing but the mechanism needed rethinking.

---

## v0.2 — Template library and the graph detour

**Goal:** Validate the template library idea under strict structural constraints.

The template library turned out to be harder to implement than expected. While working through it, I started using API models to generate code directly.

That's when the real problem surfaced: **the generated code worked but I couldn't read it.** The output had no discernible structure and no obvious entry point for changes. Frontend code was the clearest example, but the same issue appeared in any multi-component project.

Tried a detour: a Scratch-like graphical compiler where you define operations visually and preserve the underlying code files. Got far enough to define the primitives. Problems: harder to use than Mermaid, and no way to validate whether the output was actually good.

**What I learned:**
- Graph representations are readable but lower information density than code — not better for generation.
- The template library idea isn't wrong, just unsolved. Shelved, not abandoned.
- The problem I actually needed to solve was legibility, not generation.

**Goals for v0.3:**
1. For code I can't read: visible state, change impact indicators, reorganization by structure.
2. For generation: flowchart input → block-by-block generation with progressive guidance.

---

## v0.3 — Tests as intermediate language

**Hypothesis:** Use tests as the intermediate language between human intent and generated code. Tests are human-readable, follow a consistent format, and a passing suite means the code is correct by definition.

Handed the build to an AI workflow and let it run overnight.

Woke up to a bloated codebase, unnecessary features, and a test suite full of stubs — tests that verified nothing but made coverage numbers look good.

Scrapped the implementation. More importantly, realized the hypothesis was also wrong. **Tests can't fully specify behavior for event-driven systems.** Async state, interaction sequences, side effects across components — there's no assert for emergent runtime behavior. The intermediate language idea broke down exactly where I needed it most, and the same limitation applies to any event-driven codebase, not just frontend.

**What I learned:**
- Don't let AI workflows run unsupervised on open-ended tasks.
- Tests are a valid constraint for pure logic but not a general-purpose intermediate language.
- The real problem was still unsolved: how do you make the implicit structure of code visible without running it?

---

## The reframe

Stopped trying to build and started trying to think.

The real problem with spaghetti code isn't complexity. It's that **the access contract for every piece of state is implicit.** There's no record of who can write it, who reads it, or what breaks when it changes.

Wrong question: *How do I generate better code?*
Right question: *How do I make the hidden structure visible?*

This led to two ideas:

**Access Contract** — extract writers and readers for every piece of state statically, make them explicitly queryable, and flag violations.

**Virtual Files** — a projection layer over real code. Reorganize by access contract. Edit in the virtual layer; patch back to real files. If real files change externally, regenerate automatically.

Neither requires rewriting the existing code. Both work on spaghetti as-is.

---

## What's still open

**The mindmap → code problem** (originally problem 2) is now Q2 in the roadmap. The access contract system is a prerequisite — you can't constrain generation without knowing the structural boundaries first.

**WS Scheduler** — the idea that view scheduling should minimize context reloading rather than maximize information shown. Based on Working Set.

---

## What I'd do differently

To be continued...
