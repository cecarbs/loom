# pi-rust — Implementation Plan & Design Doc

> Companion to `pi-rust-proposal.md`. The proposal argues *why*; this argues *what* and *in what order*.
> Status: **planning** — no implementation started. Decisions marked **[DECIDED]**, **[SPIKE]**, or **[OPEN]**.

> **📋 For reviewers.** This is a planning/design doc — **no code has been written yet**. We want your input specifically on:
> 1. The **outstanding decisions in §10** — several need a call before implementation starts (each lists options + a recommendation + a blank line to record your decision).
> 2. The **first concrete step — the M0 language spike (§8.1)** — sanity-check the method and the decision rule before we run it.
>
> Rationale/background lives in the companion `pi-rust-proposal.md`. Suggested skim order: **§1** (what we're building) → **§3** (architecture) → **§8 + §8.1** (milestones & the first step) → **§10** (decisions we need you on).

## 1. What we're building (v1)

A Rust-native, permission-native reimplementation of the Pi coding-agent harness. **v1 target = "daily-driver":** a `ratatui` TUI agent you'd actually use day-to-day, with a mandatory permission gate, custom/local LLM providers, and a hot-reloadable extension system.

**Context:** internal work/team tool. Two hard constraints follow from that:
- The trusted core must be **small and separately auditable** (survives a real security review).
- **No npm; Cargo only** — and we keep the dependency set tight and honest (see §11 on the supply-chain framing).

### Feasibility: green

Everything load-bearing has been verified:
- **Pi is MIT-licensed** → reimplementing is legally unencumbered. Two Rust ports already exist (`nktkt/pi` ports runtime + multi-provider client + CLI) — reference implementations we can read.
- **Rhai's sandbox is real**: no fs/os/net by default, configurable resource governors (`set_max_operations`, call depth, collection sizes), no C toolchain, compile-once/eval-many for hot reload, trivial host-fn registration. Verified against the Rhai book.
- **`ratatui`** is the de-facto standard TUI lib (mature, huge adoption).
- The one weak spot in the proposal — leaning on the early-stage **`ai`** crate — is avoided: we hand-roll a thin client (§5.2).

The real risk is **not** "can this be built." It's (a) the extension-language authorability bet, and (b) over-scoping v1. Both are addressed below.

## 2. Goals / Non-goals

**Goals (v1)**
- Mandatory, non-bypassable permission gate in front of every fs/process/network capability.
- Core tools: read, write, edit, bash, plus grep/ls/glob (Pi's real default surface is ~7–9 tools, not 4).
- Thin multi-protocol LLM client: **OpenAI-compatible + Anthropic** wire protocols, streaming, tool calls.
- **Custom providers & models via config**, incl. **per-model thinking/effort levels surfaced in the UI**.
- Hot-reloadable extension system (language chosen by spike — §7).
- Usable `ratatui` TUI.
- Persistent per-project memory + remembered permission decisions.

**Non-goals (v1 → deferred)**
- Frontier-model / hosted-provider polish — **v2** (per decision). v1 targets custom/local/OpenAI-compatible endpoints.
- MCP client (fast-follow — has a trust-model wrinkle, §7.5).
- Extension distribution/registry, lockfiles (fast-follow).
- WASM stronger-isolation tier (later; the gate design leaves room for it).
- Multi-surface (desktop/VS Code/HTTP) — architecture won't preclude a client/server split, but we don't build it.
- Windows support — **[OPEN]**, likely out (bash tool + unix assumptions).

## 3. Architecture & crate layout

Mirror Pi's own modular split (`pi-ai` / `pi-agent-core` / `pi-tui`) so the **trusted core is a small set of crates you can audit in isolation**, with the untrusted surface (extensions, later MCP) clearly outside it.

```
pi-rust/                      (cargo workspace)
├─ crates/
│  ├─ pir-permit   ← permission gate + policy engine. Tiny. Zero I/O of its own.
│  ├─ pir-tools    ← core tools (read/write/edit/bash/grep/ls/glob). Every op goes through pir-permit.
│  ├─ pir-llm      ← thin client: provider/model config, protocol adapters, streaming, tool-calls, thinking levels.
│  ├─ pir-agent    ← agent loop: history, tool dispatch, context mgmt/compaction, streaming orchestration.
│  ├─ pir-ext      ← extension engine: host-fn bindings (all gated), hooks, loader, reload. [language = SPIKE]
│  ├─ pir-tui      ← ratatui frontend.
│  └─ pir          ← the binary: config load, wiring, session mgmt.
└─ ...
```

**Trust boundary:** `pir-permit + pir-tools + pir-llm + pir-agent + pir` = the trusted, compiled core. `pir-ext` scripts (and later MCP servers) are **untrusted** and reach the world *only* through gated host functions exposed by `pir-ext`. This is the proposal's "boundary fixed at compile time" made concrete as a crate boundary.

**The non-bypassable invariant (must be tested, not assumed):** every fs/process/network syscall in the whole workspace flows through a `pir-permit`-gated wrapper. Enforced by (a) `pir-tools`/`pir-ext` being the *only* crates that touch `std::fs`/`std::process`/net, (b) a CI check (grep/`clippy` lint / `cargo-deny`-style rule) that fails if those APIs appear outside the sanctioned wrappers, and (c) a test that asserts an ungated call is impossible to construct. The guarantee is only as strong as this invariant — so we make the invariant a first-class, tested artifact.

## 4. The trusted core

### 4.1 Agent loop (`pir-agent`)
Message history + tool-call/result interleaving + streaming. This — not the gate — is the real engineering. Design priorities:
- Provider-agnostic internal message model; adapters in `pir-llm` translate to/from OpenAI/Anthropic wire formats.
- **Context management / compaction** is where agents live or die (proposal gap #4). Strategy: pin system prompt + project memory; summarize/evict oldest turns as we approach the model's context window; preserve tool-result salients. Invest here early; reference `nktkt/pi` and `genai` for prior art.
- Tool dispatch routes *every* call — built-in or extension-registered — through `pir-permit` before execution.

### 4.2 LLM client (`pir-llm`) — hand-rolled thin client **[DECIDED]**
We do **not** adopt `rig-core` (large dep tree — audit surface working against the security pitch) or the early-stage `ai` crate. A thin client for two wire protocols is a few hundred lines and keeps the trusted core minimal.
- Protocol adapters: `openai` (chat/completions, SSE streaming, tool-calls; covers OpenAI-compatible + local endpoints like Ollama/vLLM/llama.cpp) and `anthropic` (messages API, streaming, tool_use).
- Deps floor: `reqwest` (or `hyper`+`rustls`) + `serde`/`serde_json` + `tokio`. Honest, standard, auditable — *not* "just tokio" (correcting the proposal's step-1 claim).
- Read `genai` and `nktkt/pi` as references; don't inherit their trees.

### 4.3 Provider & model configuration (incl. thinking levels) — align with Pi's real schema **[DECIDED]**

Confirmed against Pi's source (`packages/ai/src/types.ts`, `models.ts`; `packages/coding-agent/docs/models.md`). Pi's mechanism already produces exactly the per-model-variable behavior you want, so we **adopt its schema** (→ free import of existing Pi `models.json`) rather than invent a parallel one.

**Format:** Pi config is JSON at `~/.pi/agent/models.json`, re-read on each model switch (no restart). Shape: `{ "providers": { "<id>": ProviderConfig } }`. We define the equivalent Rust structs and deserialize with `serde`, so we read **either** Pi's `.json` (import-compat) **or** a TOML equivalent (nicer hand-authoring) into one type — same structs, two serializers.

**Provider fields:** `name`, `baseUrl`, `apiKey`, `api`, `headers`, `authHeader`, `models[]` (+ `oauth`, `modelOverrides`, `compat` later). The protocol discriminator is **`api`** — an enum string (`openai-completions`, `anthropic-messages`, `openai-responses`, …), *not* an openai/anthropic boolean. **v1 implements `openai-completions` + `anthropic-messages`.**

**Secret resolution** (mirror for import-compat): `$ENV`/`${ENV}` interpolation, `!command` (exec, use stdout), `$$`→`$`, `$!`→`!`, else literal. ⚠️ **Security note:** `!command` runs a shell command at request time to produce a secret — that's a process-exec, so it must route through `pir-permit` like any other. A naive importer would silently grant an ungated exec; we don't.

**Model fields:** only `id` is required; the rest default (`name`→`id`, `reasoning`→false, `contextWindow`→128k, `maxTokens`→16384, `cost`→zeros, `api`→provider's). Plus `cost.{input,output,cacheRead,cacheWrite}` (per-million tokens), `input: ["text"|"image"]`, and the thinking map below.

**Thinking levels — your "N levels for model A, M for model B, UI reflects it" requirement, done via masking a fixed vocabulary:**
- Fixed universe of 6 levels: `off, minimal, low, medium, high, xhigh`.
- Each model has `thinkingLevelMap: { <level>: string | null }`, **tristate per level**: key omitted = supported (provider default token); string = supported (send this literal token to the provider); `null` = unsupported (hidden).
- The **exposed set is derived per-model** by `getSupportedThinkingLevels(model)`: `off/minimal/low/medium/high` show by default, `xhigh` is opt-in (only if keyed), any `null` hides that level. So model A can expose 5, model B 3 — and **the TUI renders exactly that derived set** (§8). That *is* "the UI reflects per-model level count," natively.

  ```jsonc
  // model that only exposes off/high/xhigh, with a custom "max" token for xhigh:
  { "id": "deepseek-v4-pro", "reasoning": true,
    "thinkingLevelMap": { "minimal": null, "low": null, "medium": null, "high": "high", "xhigh": "max" } }
  // model whose thinking can't be disabled (off hidden):
  { "id": "always-on", "reasoning": true, "thinkingLevelMap": { "off": null } }
  ```
- **Provider mapping** (the two adapters we build): `openai-completions` → `reasoning_effort = thinkingLevelMap[level] ?? level` (`off` ⇒ omit the field). `anthropic-messages` → *adaptive* models send an **effort** string; older *budget* models send `thinking.budget_tokens` (defaults `minimal 1024 / low 2048 / medium 8192 / high 16384`, expanding `max_tokens` to fit).

**Verdict:** adopt Pi's schema. It satisfies the requirement, is battle-tested, and buys drop-in import of Pi provider configs. We do **not** need arbitrary per-model labels; if ever wanted, an optional per-level label override is a small additive change (fixed descriptions live in one table today).

**Adapter contract:** `pir-llm` normalizes both wire protocols to a neutral event stream — mirror Pi's `AssistantMessageEvent` (`start / text_* / thinking_* / toolcall_* / done / error`); that enum is the abstraction `pir-agent` consumes. Mirror Pi's `Usage` + `calculateCost` for a live cost readout.

### 4.4 Core tools (`pir-tools`)
read, write, edit, bash, grep, ls, glob. Each is a thin wrapper that calls a `pir-permit` sink before doing work. No tool touches the filesystem/process table without a gate call.

### 4.5 Permission gate (`pir-permit`) — the differentiator

Pi already has an *example* `permission-gate.ts` and a community `pi-permission-system`. Our delta is **structural, non-bypassable, on-by-default** — enforced by the crate boundary + invariant test, not an opt-in extension running with full ambient authority.

**Sinks** (the only ways to touch the world): `fs_read(path)`, `fs_write(path)`, `process_exec(argv)`, `net(host, port)`. Policy resolves each to `allow | deny | ask`.

**Policy** — declarative TOML, per-persona, granularity: per-capability, per-path-prefix (canonicalized), per-command, per-host. `ask` decisions can be **remembered per project** (persisted), à la Claude Code muscle memory.

**Design decisions that keep the gate honest (from review):**
- **Bash allow-rules are convenience, not a security boundary. [DECIDED]** Shell is unparseable for security (`git; rm -rf ~`, `$(...)`, `&&`, `bash -c`, symlinks). Default posture: any bash command not exactly matched → `ask`. We parse the command only to *display* the program; we treat the full string as opaque for trust. We never claim pattern-matching is airtight. (Stronger option — sandboxed/containerized exec — is a later tier, not v1.)
- **Path safety:** canonicalize + resolve symlinks; enforce project-root containment; reject `..` escapes; mindful of TOCTOU on write.
- **Network:** gate on resolved host; v1 = ask-on-new-host. Deny link-local/cloud-metadata (`169.254.169.254`) by default. Redirect/DNS-rebinding/SSRF are acknowledged limits, not solved in v1 — documented, not oversold.
- **Secrets:** process env and API keys are **never** exposed to `pir-ext` or placed in model context. Keys are read by `pir-llm` only, scrubbed from anything the model sees. (Prompt-injection defense = the gate + not exposing dangerous host fns + human-in-loop on new capabilities. Rhai's resource governors stop runaway compute, *not* exfiltration — we don't conflate the two.)

## 5. Extension system (`pir-ext`)

**Language: [SPIKE] — decided by measurement, not a priori (see §9 M0).** The proposal picked Rhai on security grounds, which are real and verified. But the *primary author is the model*, and model authoring fluency (Rhai ≪ Lua ≪ JS in corpus presence) is the bigger, harder-to-reverse risk. M0 measures first-try/third-try authoring success across **Rhai / Lua (`mlua`) / embedded JS (`boa`, pure-Rust)** and picks on evidence. Default lean stays Rhai unless the spike shows unacceptable authoring failure.

Regardless of language, the mechanics:
- **Host functions are the only capabilities**, each a thin *gated* wrapper (e.g. `http_get(url)` → `pir-permit.net(...)`). A script can never call something we didn't expose.
- Extensions register **tools** and **hooks** (`before_tool_call`, `session_start`, custom tool registration…).
- **Reload: explicit by default (`/reload`) [DECIDED]** — a deliberate checkpoint between "model wrote an extension" and "it runs," consistent with permissions-first. Relax to auto later if desired.
- **Engine threading note:** Rhai types aren't `Send`/`Sync` by default; in the tokio loop we confine the engine to a dedicated thread/actor or enable the `sync` feature. (Applies similarly to `mlua`.) Plan for this in M3.

**Worked example we must reproduce end-to-end (M3):** model notices it can't search the web → writes an extension that registers a `web_search` tool implemented over the gated `http_get` host fn → `/reload` → gate prompts on the first real network call → tool works next turn. *(Note: current Pi ships `web_fetch` (URL fetch, not search), so frame this as "compose search over fetch," not "Pi can't touch the web.")*

**Distribution (deferred, fast-follow):** one script + TOML manifest (declared capabilities, same shape as policy) + pin-to-commit + lockfile w/ content hash; install goes *through the gate*; no transitive extension deps in v1.

## 6. TUI (`pir-tui`)
`ratatui` frontend: streaming token render, tool-call/result display, **inline permission prompts** (the gate's `ask` surfaces here), a **thinking-level selector that renders the active model's `thinking_levels`** (variable count per model), model/provider switcher, session/scrollback. This is the "daily-driver" surface and the largest single chunk of UI code — sequenced late (M4) so a usable headless core exists first.

## 7. Cross-cutting concerns
- **Persistent project memory:** a `CLAUDE.md`-equivalent (e.g. `.pi-rust/MEMORY.md` — name **[OPEN]**) auto-loaded into context. Loader designed early, feature lands M5.
- **Eval harness (from M1):** fixed repo fixtures + scripted prompts + outcome assertions, so we can detect regressions in the loop/tools. Without this we're blind on every change to an agent.
- **Config UX, sessions/history, remembered decisions:** M5 polish.

## 8. Milestones & sequencing

De-risk assumptions first, keep a usable artifact early, add the big UI surface last.

| # | Milestone | Deliverable / "done" | Notes |
|---|---|---|---|
| **M0** | **Language spike** (~1 day) | Decision memo: measured authoring success (Rhai/Lua/boa-JS) → chosen language | Gates M3. Cheapest de-risk of the biggest bet. **Detailed in §8.1** — the first concrete step. |
| **M1** | **Walking skeleton** | Workspace + `pir-permit` + gated core tools + `pir-llm` (one provider) + `pir-agent` loop, **plaintext REPL**, streaming, tool calls. Gate invariant test. Tiny eval harness. | Backbone. Usable headless. The most important milestone. |
| **M2** | **Providers & thinking levels** | Full provider/model config, multiple custom providers, OpenAI + Anthropic protocol adapters, per-model `thinking_levels` (data model + request wiring; UI comes in M4). | Aligns schema with Pi for import. |
| **M3** | **Extension engine** | `pir-ext` in the chosen language: gated host fns, tool + hook registration, explicit reload. Reproduce the web-search-over-`http_get` example end-to-end. | Handles engine-threading note. |
| **M4** | **TUI** | `ratatui`: streaming render, tool-call display, inline permission prompts, **per-model thinking-level selector**, model switcher, scrollback. | The daily-driver surface. |
| **M5** | **Persistence & polish** | Project memory file, remembered permission decisions, sessions/history, config UX. | Completes "daily-driver." |
| **M6+** | **Fast-follow / v2 seeds** | MCP client (+ its trust story), extension distribution/lockfile, WASM isolation tier, frontier-model providers, multi-surface. | Out of v1. |

**v1 "daily-driver" = M0–M5.** Realistic effort: multi-month even for a small team (the proposal's "few-week sprint" is optimistic once robust streaming + a real TUI + the policy engine + extension loader are all in). We estimate against the thin slices above, not the whole.

### 8.1 M0 — the language spike (the first concrete step, ~1 day)

**Why this is step one.** The extension language is the single riskiest, hardest-to-reverse decision in the project: the *primary author of extensions is the model itself*, and model authoring fluency varies enormously by language (JS ≫ Lua ≫ Rhai in training-corpus presence). The proposal picked Rhai on security grounds — which are real and verified — but we should not bet the "agent writes itself a tool" delight factor on an a-priori guess. So before writing any of `pir-ext`, we measure. This is decision **D0** in §10.

**Candidates**
- **Rhai** — security-first default. Safe-by-default sandbox (no fs/os/net), resource governors, no C toolchain. Verified. Smallest training corpus → the authoring-fluency risk we're testing.
- **Lua (`mlua`)** — larger corpus, better human-authorability. Opt-out safety (must strip `io`/`os`/`require`); vendors/compiles C.
- **Embedded JS (`boa`)** — maximum model fluency (JS), pure-Rust engine (no C). Lower engine maturity is the risk here.

**Method**
1. Fix ~10 representative extension tasks the model would realistically self-author: web-search-over-`http_get`, git-log summarizer, a ripgrep/file-grep wrapper tool, a JSON pretty-print tool, a `before_tool_call` hook that redacts secrets, a `session_start` banner, an HTTP POST to a webhook, a small **stateful** counter tool, a tool that **chains two host fns**, and one deliberately tricky task (error handling / async).
2. For each engine, give the model the same context the real system would: the host-fn surface + 1–2 canonical examples.
3. Have the **target model** write each task in each language. Record: **first-try** parse+run success, **third-try** success (with error feedback looped back), and a subjective correctness/idiomaticity score.
4. Log failure modes (e.g., reaching for Rust-isms or JS-isms that don't exist in Rhai; forgetting `mlua`'s value conversions; boa runtime gaps).

**Decision rule.** Pick the language with the best combined first-try + error-recovery authoring success, weighted toward the safer sandbox. **Rhai wins ties** (safe-by-default). Switch away from Rhai only if its authoring success is *materially* worse than a candidate that clears the bar comfortably. (Reviewer: is this the right rule, or should security get a stronger/weaker weight?)

**Deliverable.** A short decision memo (appended to this doc or linked) with the numbers and the chosen language. Resolves **D0** and unblocks **M3** (`pir-ext`).

**Dependencies.** M0 depends on **none** of the §10 decisions D1–D5 — it can run in parallel while those are settled with the team. It decides only D0.

## 9. Top risks & mitigations

| Risk | Mitigation |
|---|---|
| **Model can't author extensions fluently in Rhai** (biggest, hardest to reverse) | M0 spike measures it; boa-JS/Lua are live alternatives; ship canonical extension examples in context regardless. |
| **Bash "permissions" give false security** | Treat bash allow-rules as convenience; `ask` on anything unmatched; never claim the matcher is a boundary; sandbox-exec as a later tier. |
| **Security-review pitch gets picked apart** | Honest framing: Cargo shifts ACE to *build*-time (`build.rs`/proc-macros), doesn't eliminate it; tight, auditable dep set; gate invariant is *tested*. |
| **Scope/timeline blowout** | Thin vertical slices; M1 usable headless; TUI deferred to M4; MCP/distribution/WASM explicitly out of v1. |
| **Streaming + compaction + coherence** (the actual hard engineering) | Invest design in `pir-agent` early; reference `nktkt/pi`, `genai`; eval harness from M1. |
| **MCP punches a hole in the permission model** | Deferred; when added, MCP calls route through the gate and the server process is treated as a native trust boundary (not a sandboxed extension). |

## 10. Outstanding decisions (reviewer input requested)

These are the decisions that gate implementation. Each has options, a recommendation, and a blank line to record the call. **Reviewer:** please mark a decision (or push back on the recommendation) on each. Status legend: `OPEN` = needs a call · `PENDING SPIKE` = M0 will decide · `RESOLVED` = decided, confirm you agree.

| ID | Decision | Status | Needed by | Owner |
|---|---|---|---|---|
| D0 | Extension language | PENDING SPIKE (§8.1) | M3 | — |
| D1 | Permission-policy file format | OPEN | M1 | — |
| D2 | Bash isolation in v1 | OPEN | M1/M2 | — |
| D3 | Windows support | OPEN | M1/M4 | — |
| D4 | Project-memory filename | OPEN | M5 | — |
| D5 | Persona model | OPEN | M2/M5 | — |
| C1 | Provider/model config schema | RESOLVED (confirm) | M2 | — |

**D0 — Extension language** · *decided by the M0 spike, not by discussion*
- **Options:** Rhai (security-first, safe-by-default, smallest corpus) · Lua/`mlua` (bigger corpus, opt-out safety, C) · embedded JS/`boa` (max model fluency, pure-Rust, less mature).
- **Recommendation:** run the **M0 spike (§8.1)** and pick on measured authoring success; Rhai is the default that must be *beaten*.
- **Impact:** high, hardest-to-reverse. Determines the entire `pir-ext` implementation.
- **Decision:** ________________________________________

**D1 — Permission-policy file format** · OPEN
- **Options:** (a) **TOML** — consistent with the Cargo/crates world. (b) YAML — as written in the original proposal.
- **Recommendation:** TOML.
- **Impact:** low/mechanical; touches only the `pir-permit` policy loader. *(Provider config is settled separately — see C1.)*
- **Decision:** ________________________________________

**D2 — Bash isolation in v1** · OPEN · *the one with real scope implications*
- **Options:** (a) **ask-on-unmatched only** — bash allow-rules are convenience, not a security boundary; anything not exactly allowlisted prompts. (b) **sandboxed/containerized exec** — run bash inside an OS-level sandbox so the boundary is real, at real added cost.
- **Recommendation:** (a) for v1; design so (b) can layer on later (it's also the WASM-tier story).
- **Impact:** medium–high. (b) meaningfully enlarges v1 scope and platform assumptions. See §4.5.
- **Decision:** ________________________________________

**D3 — Windows support** · OPEN
- **Options:** in / out for v1.
- **Recommendation:** **out** for v1 (the bash tool + TUI assumptions are unix-centric); revisit post-v1.
- **Impact:** affects the bash tool (M1) and TUI (M4), and the D2 sandbox choice.
- **Decision:** ________________________________________

**D4 — Project-memory filename/convention** · OPEN
- **Options:** `.pi-rust/MEMORY.md` · `PI.md` · other.
- **Recommendation:** defer; low-stakes, decide at M5.
- **Impact:** low. The `CLAUDE.md`-equivalent auto-loaded into context (§7).
- **Decision:** ________________________________________

**D5 — Persona model** · OPEN
- **Options:** (a) single default policy in v1. (b) multiple per-persona policies (as the proposal hints).
- **Recommendation:** (a) single default for v1; the policy engine can grow personas later.
- **Impact:** low–medium; affects `pir-permit` policy scoping.
- **Decision:** ________________________________________

**C1 — Provider/model config schema** · RESOLVED (confirm)
- **Resolution:** adopt Pi's `models.json` JSON schema (drop-in import of existing Pi configs) via `serde`, and also accept a TOML serialization of the same structs for hand-authoring. Details + the thinking-level mechanism in **§4.3**.
- **Reviewer:** confirm you're happy mirroring Pi's schema (incl. the fixed 6-level `thinkingLevelMap` vocabulary), or flag if you'd rather diverge.
- **Decision:** ________________________________________

---
**Immediate next step:** the **M0 language spike (§8.1)** can start now — it depends on none of D1–D5. Settle D1–D3 in parallel (D4–D5 can wait). Then M1.
