# Handoff

This project's durable context lives in two docs, not in anyone's chat history:
`pi-rust-proposal.md` (the *why*) and `pi-rust-plan.md` (the authoritative *what/when*).
To hand the work to a fresh LLM/agent that has no prior context, paste the prompt below.

## First prompt for a continuing agent

> Assumes the agent is operating **inside this repo** (can read files). If you're handing
> off to a plain chat LLM with no file access, paste the full contents of `pi-rust-plan.md`
> (and ideally `pi-rust-proposal.md`) into the conversation instead.

```
You're picking up an in-progress project. The repo is a permission-native Rust
reimplementation of the "Pi" coding-agent harness (earendil-works/pi). All planning
is done and committed; there is NO implementation code yet beyond a hello-world Cargo
scaffold.

Before doing anything, read these two files in the repo root — they are the source of
truth, in this order:
  1. pi-rust-proposal.md  — motivation and architecture rationale (the "why").
  2. pi-rust-plan.md      — the authoritative implementation plan: goals/non-goals,
     crate layout, milestones M0–M5, and two sections to read carefully:
       • §8.1 — M0, the language spike (the first concrete step).
       • §10  — the decision log of outstanding decisions (D0–D5, C1).

Current status:
  • v1 target is a "daily-driver" TUI agent; milestones M0 → M5 are in §8.
  • Immediate next step is M0 (§8.1): measure how reliably the target model can author
    extensions in Rhai vs. Lua vs. embedded JS (boa), then pick the extension language
    on evidence. M0 depends on none of the open decisions and can start now.
  • Several §10 decisions (esp. D1 policy format, D2 bash isolation, D3 Windows) are
    gated on human input. Recommend, but do NOT unilaterally lock them.

How to work on this:
  • Do NOT write implementation code until (a) the M0 spike has chosen the extension
    language and (b) the §10 decisions gating M1 are made. If asked to build before
    then, flag it rather than proceeding.
  • Independently verify any external claim you're about to depend on (crate versions,
    APIs, licenses). The plan's claims were fact-checked on 2026-07-08, but re-check
    anything load-bearing before building on it.
  • Confirm with me before any outward-facing or hard-to-reverse action (push, publish).

First response: read both docs, then give me (1) a ~5-line summary of where the project
stands, (2) your recommendation on whether to run the M0 spike now or settle §10 first,
and (3) any gaps or risks you see in the plan. Do not write code yet.
```

## Why it's shaped this way

- **Points at the docs rather than restating them** — the plan is the durable artifact;
  the prompt just orients the agent and marks the current cursor (M0 next, §10 gated), so
  it stays valid as the plan evolves.
- **Encodes the two guardrails that matter** — plan/verify before building, and don't
  unilaterally decide the human-gated calls — so a fresh agent won't charge into
  implementation or silently pick D1–D3.
- **Asks for a conceptual read-back first**, surfacing any misunderstanding before work starts.
