---
title: "DeepSeek's Agent Framework: Even the Agent Loop Is a Plugin"
description: "A deep dive into DeepSeek Harness — 40+ packages, 20+ capability seams, and an Agent Loop that can be swapped from a YAML config. Capability Seam's three-role separation, the Session Log as the single source of truth, and a 'model-visible ⟺ logged' runtime invariant."
date: 2026-08-17
tags: ["AI Agent", "Architecture", "Developer Tools"]
---

> DeepSeek open-sourced an Agent Harness. I spent two days going through the repo — 40+ packages, 20+ capability seams — and the most notable thing: **the Agent Loop itself is a plugin, swappable from a YAML config.**

![Everything-is-a-plugin concept](/images/deepseek-harness/deepseek-harness-everything-plugin.png)

## The short version

The biggest takeaway from reading the repo: this team has a thing for plugin-izability.

Not the "we support plugin extensions" kind. The **entire product — including the Agent Loop — is assembled from plugins**. There is no privileged core whose implementation you're stuck with. Want a different LLM adapter? Write a plugin. A different filesystem backend? Write a plugin. A different Agent Loop? Write a plugin.

It sits at the opposite extreme from Pi. Pi subtracts to the bone; DeepSeek Harness maximizes replaceability. Both have trade-offs — it depends on your use case.

Three differences that matter most:

**Agent Loop**: hard-coded in Pi, hard-coded in Claude Code, replaceable in DeepSeek Harness — you swap the entire execution loop from a YAML config.

**Degree of plugin-ization**: Pi is light extension; DeepSeek Harness is everything-as-a-service, with the Agent Loop itself built from plugins.

**Self-modification**: in other frameworks, a human configures the agent; in DeepSeek Harness, the agent can mount/unmount its own plugins.

The cost of "everything is a plugin" is complexity — 40+ packages is a real learning curve. But the payoff is clear: any layer you're unhappy with can be replaced.

## The basics

| Item | Value |
| --- | --- |
| Repo | [deepseek-ai/deepseek-harness](https://github.com/deepseek-ai/deepseek-harness) |
| Version | `0.1.0-rc.5` (developer preview) |
| License | MIT |
| Runtime | Node.js 22.19+ / 24+ |
| Package manager | pnpm 11.7 workspaces |
| Underlying framework | Cordis (vendored, has a paper) |
| Package count | 40+ npm packages (`@deepseek-ai/dsh-*`) |
| Build | tsc + tsdown |
| Testing | 5 layers, including a 100% line-coverage gate |

```sh
# One line to start the Web UI, default http://127.0.0.1:3080
npx @deepseek-ai/dsh web
```

40+ packages. Not a "core + a few plugins" structure — **the whole product is decomposed into modules**. Here are the designs I found most worth studying.

## The Agent Loop is a plugin

This surprised me the most. The Agent Loop is the most "core" part of any framework — and here it's replaceable.

In most Agent frameworks the loop is hard-coded — changing execution strategy means changing core code. DeepSeek Harness makes the Loop a `ctx.agentLoop` service, on equal footing with every other service. You specify which implementation to use in `cordis.yml`.

The whole product sits on a plugin framework called Cordis (there's a paper, [_A Programming Paradigm for Spatiotemporal Composability_](https://github.com/cordiverse/paper)), built on a handful of concepts:

```mermaid
flowchart TD
    Plugin["Plugin<br/>object implementing a Service interface"]
    Context["Context<br/>service registry ctx.key"]
    Inject["Inject<br/>declarative dependency injection"]
    Events["Typed Events<br/>type-safe event communication"]
    Effects["Effects<br/>reversible registration = cleanable"]

    Plugin -->|"mounts to"| Context
    Plugin -->|"declares needs"| Inject
    Context -->|"dispatches"| Events
    Plugin -->|"registers via effect/on"| Effects
    Effects -->|"auto-cleaned on unmount"| Context

    style Plugin fill:#e3f2fd
    style Context fill:#e8f5e9
    style Events fill:#fff3e0
    style Effects fill:#fce4ec
```

The most interesting bit is **the reversibility of Effects**. Everything a plugin registers — tools, prompt snippets, event listeners — goes through `ctx.effect()` or `ctx.on()`, and gets cleaned up automatically on unmount. That sounds simple, but I've read a lot of Agent framework plugin systems and almost none of them actually do it — either the registration lives forever, or unmounting leaves state behind. Cordis provides the effect mechanism at the base layer; DeepSeek Harness is the first framework I've seen apply it at the Agent-framework level: registration is a side effect, unmount is cleanup, guaranteed at the framework level.

![Cordis plugin architecture](/images/deepseek-harness/deepseek-harness-cordis-architecture.png)

Extending it doesn't require editing source or writing monkey patches:

- Swap the LLM adapter → register a new `ctx.llm` provider
- Swap the filesystem backend → swap the `ctx.fs` provider (local → E2B sandbox)
- Swap the Agent Loop → swap `ctx.agentLoop`
- Add an approval policy → listen to `agent/*` events

## Capability Seam: three-role separation

"Adding a capability" in most frameworks means writing a function and registering it in a tool table. When something breaks you dig through source to find the interface, and switching backends means touching the Consumer.

DeepSeek Harness formalizes "capability" into three roles, called a Seam:

```mermaid
flowchart LR
    SD["Service Definition<br/>declares interface + ctx key<br/>(abstract class / registry)"]
    SP["Service Provider<br/>implements interface<br/>(can be several)"]
    Consumer["Consumer<br/>uses interface<br/>(usually a model tool)"]

    SD -->|"implemented by"| SP
    SD -->|"injected into"| Consumer
    SP -->|"supplies"| Consumer

    style SD fill:#e3f2fd
    style SP fill:#e8f5e9
    style Consumer fill:#fff3e0
```

Taking the Shell capability as an example:

| Role | Package | Responsibility |
| --- | --- | --- |
| Service Definition | `dsh-shell` | Declares the `ctx.shell` interface and types |
| Service Provider | `dsh-bash-local` | Local bash execution |
| Service Provider | `dsh-bash-sandbox` | Sandboxed bash execution |
| Consumer | `dsh-tool-bash` | The bash tool the model sees |

Swapping a Provider doesn't touch the Consumer. To go from local execution to an E2B remote sandbox, swap one provider package — the tool definition stays exactly as it is.

The repo has 20+ such seams:

| Domain | Seam | Provider count |
| --- | --- | --- |
| Model | `ctx.llm` | 3 (DeepSeek / pi-agents / replay) |
| Filesystem | `ctx.fs` | 3 (local / e2b / sandbox) |
| Subprocess | `ctx.subprocess` | 2 (local / e2b) |
| Shell | `ctx.shell` | 2 (bash-local / bash-sandbox) + pwsh |
| Terminal | `ctx.terminals` | 1 (terminal-bash) |
| Subagents | `ctx.subagents` | 6 (spawn/fork/sdk/ACP/Codex/Claude Code) |
| Sandbox | `ctx.sandbox` | 3 backends (bwrap / Landlock / Seatbelt) |
| Web | `ctx.web` | 4 (Exa / Perplexity / DeepSeek search / HTTP fetch) |
| LSP | `ctx.lsp` | 1 (stdio) |
| Persistence | `ctx.sessionPersistence` | 2 (JSONL / SQLite) |

The 6 subagent backends are worth noting — including providers that talk directly to Codex and Claude Code. The sandbox also comes in three flavors (bwrap / Landlock / Seatbelt), covering Linux / macOS's newer security module / the classic macOS sandbox.

## Session Log: what the model sees must be in the log

What does the model actually see during a run? Most frameworks only keep a prompt log; session history, tool calls, and context injection are scattered around.

DeepSeek Harness builds an append-only event stream called the Session Log, and derives every fact from it:

```mermaid
flowchart TD
    subgraph SessionLog["Session Event Log"]
        direction TB
        E1["user/message"]
        E2["assistant/chunk"]
        E3["assistant/message"]
        E4["tool/call"]
        E5["tool/result"]
        E6["turn/start · turn/end"]
        E7["step/start · step/end"]
    end

    Model["model request"] -->|"deriveMessages()<br/>built-in message derivation"| SessionLog
    UI["Web UI rendering"] -->|"session/event"| SessionLog
    Persist["persistence<br/>JSONL / SQLite"] -->|"write"| SessionLog
    Fork["Fork / Resume"] -->|"copy from"| SessionLog
    Telemetry["telemetry"] -->|"derive"| SessionLog

    style SessionLog fill:#e3f2fd
```

There's one rule that's enforced especially hard: **model-visible must be logged**.

This isn't a "suggestion" — there's a runtime invariant assertion in the code. If something reaches a model request but can't be rebuilt from the session log, it errors out.

Practical payoff: you changed a context-injection rule — how do you confirm the model actually saw it? Check the log. No corresponding event in the log, and the model definitely didn't see it. Fork, Resume, telemetry — all derived from that one stream, no second set of state to maintain.

Pi takes the stateless route; DeepSeek Harness takes "everything is state, and all state lives in the log." Both work, but I lean toward DeepSeek's — **the "model-visible ⟺ logged" invariant is the single most portable Agent design principle I've come across**. It turns "what the model sees must be traceable" from a best practice into a runtime check.

## Profile + Bundle + Patch: configuration as composition

Agent run modes differ a lot: the Web UI needs a server plus frontend, Headless is a one-shot runner, ACP speaks JSON-RPC. Most frameworks either hard-code a few modes or cobble them together from env vars.

DeepSeek Harness uses three layers:

```mermaid
flowchart TD
    Profile["Profile<br/>named composition<br/>(web / headless / acp)"]
    Bundle["Bundle<br/>installable plugin layers<br/>(base / web-app / headless)"]
    Patch["Patch<br/>config overrides<br/>(cordis.patch.yml)"]

    Profile -->|"includes multiple"| Bundle
    Profile -->|"stacks"| Patch

    Base["dsh-base<br/>model + tools + persistence + sandbox"]
    WebApp["dsh-web-app<br/>browser app"]
    Headless["dsh-headless<br/>one-shot runner"]

    Bundle -.-> Base
    Bundle -.-> WebApp
    Bundle -.-> Headless

    style Profile fill:#e3f2fd
    style Bundle fill:#e8f5e9
    style Patch fill:#fff3e0
```

- **Profile** — a named composition; `web` and `headless` ship out of the box
- **Bundle** — a distributable plugin layer; a profile stacks multiple bundles
- **Patch** — a YAML override; replace any config line or insert new ones

Boot order: Bundle → profile-level patch → home-level patch → `--patch` CLI override.

```sh
# Inspect the actual config tree that boots
dsh --profile web --dump-config
```

Any line of the output can be replaced by a patch. Editing YAML has a lower bar than editing code, so operators can tune Agent behavior without touching source.

## The Agent event flow

A single agent interaction splits into two layers, Turn and Step:

- **Turn** — one complete input consumption, from model through tool execution to the end
- **Step** — one model request plus the tool calls it triggers

```mermaid
sequenceDiagram
    participant User as User/Input
    participant AgentLoop as Agent Loop
    participant Prompt as Prompt Assembly
    participant LLM as LLM Adapter
    participant Tools as Tool Pipeline

    User->>AgentLoop: turn/start
    AgentLoop->>AgentLoop: claim input + queue messages
    AgentLoop->>Prompt: assemble prompt sections + tool schemas
    AgentLoop->>AgentLoop: agent/pre-step (waterfall)
    AgentLoop->>LLM: step/start → agent/request → llm/stream
    LLM-->>AgentLoop: assistant/chunk* → assistant/message
    AgentLoop->>Tools: tool/call → pre-execute → execute → post-execute → result
    Tools-->>AgentLoop: tool/result
    AgentLoop->>AgentLoop: step/end
    alt more tool requests or new input
        AgentLoop->>AgentLoop: next step
    end
    AgentLoop->>AgentLoop: agent/turn-stopping (serial)
    AgentLoop-->>User: turn/end
```

Events live in three domains:

| Event domain | Persisted? | Purpose |
| --- | --- | --- |
| `session/*` | ✅ written to log | facts that survive restarts |
| `agent/*` | ❌ real-time | observe/intercept a live Agent |
| Capability-domain events | ❌ real-time | attach policy and adapters to seams |

Waterfall events (`agent/pre-step`, `agent/request`, `llm/stream`, `tools/*`) are around-middleware — listeners must delegate via `next()`, and the chain short-circuits if they don't. Same pattern as Express/Koa middleware, but type-safe.

## Docs-as-code: a consistency gate between docs and code

In most projects, docs and code are two separate things that drift apart over time. DeepSeek Harness's answer isn't "people should be disciplined" — it's **making doc consistency a CI gate**:

A type definition pasted into the docs doesn't match the source? CI fails. A broken link? CI fails. An exported function missing JSDoc? CI fails. Docs over the word budget? CI fails.

Any non-trivial code change must ship with an Agent Note (a design-decision doc) in the same PR. Not a suggestion — a gate. No note, no merge.

Honestly, that level of quality control is rare in open source.

## Where this is worth referencing

1. **Building a highly-customizable Agent platform** — the three-role capability seam separation ports directly
2. **Need multi-backend switching** (local / sandbox / remote) — the Provider pattern is the standard answer
3. **Session reliability matters** — the "model-visible ⟺ logged" invariant
4. **Team-grade agents need access control** — approvals + sandbox backends combined

## Where it's not a fit

1. **Lightweight personal tools** — Pi's "light Harness" fits better; 40+ packages is over-engineering for a personal tool
2. **Rapid prototyping** — the declarative config + three-layer composition learning curve doesn't suit fast trial-and-error
3. **Non-TypeScript stacks** — it's deeply tied to the TypeScript type system; porting cost is high

## Directly reusable design principles

1. **Build capabilities as three-role seams** — split Service Definition / Provider / Consumer so swapping a Provider never touches the Consumer
2. **Make the session log the single source of truth** — what the model sees must be reconstructible from one log stream
3. **Registration as side effect, unmount as cleanup** — don't manage lifecycle by hand
4. **Keep config declarative** — YAML composition over code if-else
5. **Turn "model-visible ⟺ logged" into a runtime assertion** — a gate, not a guideline

## Closing

Two directions: Pi is "cut what you can", DeepSeek Harness is "split what you can". What they share is having thought hard about "what belongs in the framework and what doesn't" — Pi's answer is "as little as possible", DeepSeek's is "everything pluggable".

Which path you take depends on the scenario. But if you're building an Agent platform, three design ideas are worth serious consideration: the Capability Seam's three-role separation, the Session Log as the source of truth, and a replaceable Agent Loop.

---

## References

- [DeepSeek Harness repo](https://github.com/deepseek-ai/deepseek-harness) — verified: README, AGENTS.md, architecture.md, development.md, cordis-primer.md, packages/README.md
- [Cordis framework](https://github.com/cordiverse/cordis)
- [Cordis paper: A Programming Paradigm for Spatiotemporal Composability](https://github.com/cordiverse/paper) (not line-by-line verified)
- Comparison reference: [Pi Agent Design Philosophy](/posts/pi-agent-design-philosophy)
