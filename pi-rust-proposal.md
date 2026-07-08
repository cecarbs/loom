# Pi (Rust): A Permission-Native Rewrite of the Pi Coding Agent Harness

## Background

[Pi](https://pi.dev) is an open-source, minimal, terminal-first AI coding agent harness (`earendil-works/pi`). Its philosophy is the opposite of "batteries included" tools like Claude Code or OpenCode: a ~1,000-token system prompt, a four-tool core (Read, Write, Edit, Bash), and everything else — UI, workflow, orchestration — left to user-authored extensions. Its creator built it specifically to regain control over what gets injected into the model's context and how the agent behaves, rather than accepting a sealed product.

Two things about Pi are appealing for our use, and one thing is a blocker:

- **Appealing:** the minimal core + self-extension model (the agent can write its own extensions at runtime — e.g., writing itself a web-search tool it didn't originally have).
- **Appealing:** it adapts to your workflow instead of forcing you into someone else's.
- **Blocker:** it ships as an npm package with no built-in permission system — by default it runs with full permissions of whatever launched it, and third-party extensions run with full system access. Our npm policy (curated allowlist only) makes adopting the existing TypeScript implementation impractical, while Cargo is unrestricted for us.

**Proposal:** build a Rust-native reimplementation of Pi's core ideas, with two changes baked in from day one rather than bolted on:
1. A permission/capability system that is structurally part of the core, not an optional extension.
2. No npm dependency at all — Cargo only, which also incidentally eliminates an entire class of supply-chain risk (npm's install-time lifecycle-script execution has no equivalent in the Cargo ecosystem).

There is prior art suggesting this is tractable: `ai` on crates.io is explicitly "inspired by pi" and already provides a unified multi-provider LLM client with tool calling and a lightweight agent loop in Rust.

---

## Architecture Overview

| Layer | Technology | Compiled or hot-reloadable? |
|---|---|---|
| Agent loop (message history, tool-call dispatch, context compaction, streaming) | Rust (native) | Compiled — requires rebuild to change |
| Core tools: Read, Write, Edit, Bash | Rust (native), using `std::fs` / `std::process` | Compiled |
| Unified multi-provider LLM client | Rust, own thin layer or adapted from existing crates (`ai`, `llm`, `rust-genai`) | Compiled |
| **Permission gate** | Rust (native), sits in front of every tool dispatch | Compiled — non-bypassable by construction |
| TUI | `ratatui` | Compiled |
| **Extension system** | **Rhai** (embedded scripting) | **Hot-reloadable, no rebuild** |

The key architectural idea: **the trusted boundary is fixed at compile time; behavior on top of it is infinitely reconfigurable at runtime.** New primitive capabilities (e.g., "the agent can now make raw HTTP calls") require a Rust change and a rebuild. New *behavior* built from existing primitives (e.g., "the agent composes an HTTP call into a working web-search tool") is pure Rhai and never requires touching the trusted core.

---

## The Permission System

Rather than an optional extension (as in the community `pi-permission-system` project for the TS version), permission checking is a mandatory pass-through every tool call goes through, by construction:

- Every tool call — built-in (Read/Write/Edit/Bash) or extension-registered — is routed through a `PermissionGate` before execution. There is no code path that reaches the filesystem, process spawning, or network without going through it.
- Policy is declarative, YAML-based, per-agent-persona, with `allow` / `deny` / `ask` at increasing granularity (per-tool, per-bash-command-pattern, per-network-target, per-skill).
- `ask` decisions are logged and can be "remembered" per project (à la Claude Code's permission muscle memory).
- The same gate governs **extension installation** as governs **extension execution** (see Distribution, below) — one mechanism, not two bolted-together trust models.

This alone addresses the single most-requested feature gap identified when comparing Pi to Claude Code/OpenCode users: real permissions, on by default, not DIY.

---

## The Extension System: Rhai

### Why an embedded scripting language, not native Rust code

Pi's TS extensions work because JavaScript can `import()` arbitrary code at runtime with no compile step. Rust is compiled, so "drop a file in a folder and it just works" isn't free — it has to be built deliberately. Options considered:

| Option | Verdict |
|---|---|
| Dynamic library loading (`libloading`) | Rejected — unsafe, ABI-fragile, gives extensions full ambient host permissions (reintroduces the exact problem we're trying to solve) |
| WASM (`wasmtime`/`wasmer`) | Strong for untrusted/community code — true capability-based sandbox, but adds a compile step, slowing the "agent writes itself a tool in real time" loop |
| Subprocess / JSON-RPC (LSP-style) | Airtight OS-level isolation, but kills the lightweight "drop a script and go" authoring experience |
| Embedded JS (`rquickjs`/`boa`) | Best compatibility with the *existing* Pi TS extension ecosystem and LLM familiarity with TS/JS, but reintroduces either a C dependency (`rquickjs`) or a less mature engine (`boa`) |
| **Embedded scripting: Rhai or Lua (`mlua`)** | **Chosen approach** |

### Rhai vs. Lua, decided in favor of Rhai

Given that the primary author of extensions is **the model itself, writing self-extensions at runtime** (not humans), the deciding factor is the sandbox's default posture:

- **Rhai ships nothing dangerous by default.** No file I/O, no OS calls, no networking in its standard library — capabilities are opt-in, one Rust function registration at a time. A missing capability is a safe failure.
- **Lua's stdlib ships `io`, `os`, `require` on by default** — `mlua` can strip these to a safe subset, but that's an opt-out step you must remember to do and keep correct over time.
- Rhai has built-in resource governors (max operations, call-stack depth, collection sizes) — protects against runaway or adversarial scripts (e.g., from something the model read via prompt injection) with no extra engineering.
- Rhai is pure Rust — no C toolchain to link, cleaner story for a security review than vendoring and compiling a C interpreter (as `mlua` does even in vendored mode).
- Host-side interop (exposing a Rust function to a script) is close to boilerplate-free in Rhai, lowering *our* implementation cost for each new bound primitive.
- Tradeoff: Rhai is a smaller, more bespoke ecosystem than Lua. If we ever want humans (not just the model) to hand-write or debug extensions, Lua's much larger footprint (games, Neovim, etc.) would make it the easier language to onboard people into.

**Decision: Rhai**, given the model-as-primary-author assumption. Revisit if human-authored extensions become common.

### How hot-reload actually works (no recompilation)

Rhai scripts are not compiled into the binary. The Rust binary links the `rhai` crate once and instantiates an `Engine` with our bound host functions (Read/Write/Edit/Bash primitives, permission-gated). At runtime:

1. Pi scans the extensions directory for `.rhai` files.
2. Each file is parsed into an in-memory AST via `engine.compile_file(...)` — parsing, not native compilation. Milliseconds.
3. Hook points (`before_tool_call`, `session_start`, custom tool registration, etc.) invoke the relevant AST via `engine.eval_ast(...)`.
4. If a file changes (the model wrote or edited one), Pi re-parses just that file and swaps the AST. **The host process itself never rebuilds or restarts.**

Reload can be explicit (`/pi ext reload`, or a tool call the model invokes) or implicit (mtime check before each hook fire). **Recommendation: explicit by default** — gives a checkpoint between "model wrote an extension" and "that extension starts executing," consistent with our permissions-first philosophy. Can be relaxed later.

### Worked example: the web-search extension

This traces the exact scenario that motivated this project (Pi's real inability to search the web out of the box):

1. **Prerequisite (one-time, requires a rebuild):** we bind a host function, e.g. `http_get(url) -> Result<String, _>`, into the `Engine` ahead of time. This is the only part that's Rust and compiled — a script can never call a capability we haven't explicitly exposed.
2. **The model notices it has no web-search tool**, writes `~/.pi/extensions/web_search.rhai`, which registers a new tool via our tool-registration hook, and implements the handler by calling `http_get` against a search API and formatting the JSON response.
3. **Pi reloads the extension** (explicit or automatic) and parses it into an AST.
4. **`web_search` is now available to the model on every subsequent turn, in this and future sessions — no recompilation.**
5. **The permission gate intercepts the first real network call** the script makes (its manifest declares a `network` capability) and prompts for approval, exactly as it would for a native tool — one permission mechanism governing both built-in and extension-added capabilities.

The result: the same self-extension delight factor Pi's TS version has today, but "the model wrote itself a web search tool" and "the model wrote itself something with unrestricted system access" are now structurally different outcomes — not just different intentions — because the ceiling is whatever we deliberately exposed to the `Engine`, by construction.

---

## Distribution: Sharing Extensions via GitHub/Bitbucket

Community/team-shared extensions need a lighter-weight answer than npm, consistent with the "flat, auditable trust graph" goal:

- **Extension = one script file + one manifest** (TOML): name, version, hook(s) it registers for, and a declared capability list (`fs:read`, `bash:git *`, `network: <domain>`, etc.) — the same shape as the native permission policy.
- **Pin to a commit, not a branch.** `pi ext install github.com/user/repo@<sha>`, fetched as a tarball via the provider's API (no dependency on `git` itself).
- **Lockfile** (`pi-extensions.lock`): extension name → repo URL → pinned SHA → content hash. Same reproducibility guarantee Pi already values; tampering is detectable via hash mismatch.
- **No transitive extension dependencies (v1).** Extensions depend only on capabilities *we* expose, not on each other — sidesteps dependency-resolution complexity entirely and keeps each extension's blast radius fully described by its own manifest.
- **Installation itself goes through the permission gate** — the manifest's declared capabilities are resolved against policy exactly like a live tool call, `ask` by default, `allow` for a team-maintained vetted list.

Because extensions are readable source (Rhai text), not compiled artifacts, they're diffable and reviewable — by a human or by having an agent review the script before approving installation — which is a real advantage over binary plugin formats for this specific "pull code from a stranger's repo" pattern.

---

## Feature Gaps to Address (relative to Claude Code / OpenCode / Codex)

Identified from comparative research, roughly in priority order for us:

1. **Real permissions, on by default** — addressed by design above.
2. **MCP support** — Pi has none built-in; worth adding to the native core if we need to reach internal tools (Jira, wikis, etc.) via MCP servers.
3. **Persistent project memory** (a `CLAUDE.md`-equivalent) — Pi handles this via manual markdown conventions today; worth formalizing.
4. **Cross-turn coherence on long multi-step tasks** — an agent-loop/context-management design property, not a bolt-on feature; needs deliberate attention during the rewrite.
5. **Multi-surface access** (desktop app, VS Code extension, web/HTTP API beyond the TUI) — likely out of scope for v1, but worth an architecture that doesn't preclude a client/server split later.

---

## Suggested Build Order

1. Agent loop + permission gate + native Read/Write/Edit/Bash tools (Rust only, zero external crates beyond `tokio`, satisfies strictest security review).
2. Rhai integration: bind a minimal set of host functions, hook points, and the extension loader/reload mechanism.
3. Basic MCP client support.
4. Persistent project-memory file convention.
5. TUI via `ratatui`.
6. Extension distribution/sharing system (manifest format, pinned installs, lockfile).
7. WASM support as an optional, stronger-isolation tier for community/unreviewed extensions, layered on top of the same permission-gate concept.

**Rough scope estimate:** a genuinely complete version (core + permissions + Rhai extensions + TUI) is comparable to Pi's own TypeScript coding-agent package in scope — likely in the 15,000–30,000 LOC range for a serious v1, achievable as a multi-month solo effort or a few-week sprint for a small team.

---

## Open Questions for the Team

- Explicit vs. implicit extension reload — how much friction do we want between "model writes an extension" and "it runs"?
- Do we want Lua instead of Rhai if we expect meaningful human (not just model) extension authorship down the line?
- How aggressively do we pursue MCP support in v1 vs. treating it as a fast-follow?
- What's our policy for team-shared extensions — informal review, or a hosted internal registry with mandatory vetting before anything gets an `allow` by default?
