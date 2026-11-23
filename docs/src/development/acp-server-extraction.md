# Spec: Extracting the Native Agent as a Full ACP Server

Status: Draft
Author: Agent proposal (adapted from the `forge` ACP layering model)

## Motivation

Today Zed's built-in agent (`NativeAgent`) can only be consumed in-process by the
Agent Panel. Zed is an ACP **client** (it drives external agents like Claude Code
and Gemini CLI over the wire), but the native agent itself has no wire boundary:
`NativeAgentConnection` implements the internal `acp_thread::AgentConnection`
trait and is handed directly to the panel as an `Rc`.

Making the native agent reachable as a standalone ACP **server** (stdio, JSON-RPC)
would let any ACP client — including other editors, CLI tools, or agents such as
Claude Code spawning Zed as a sub-agent — drive Zed's agent.

## Goals

- Expose `NativeAgent` sessions over the ACP wire protocol (`initialize`,
  `session/new`, `session/prompt`, `session/load`, `session/cancel`,
  `session/update` streaming, permission and elicitation round-trips).
- Keep the extraction isolated in a new crate with a narrow dependency port onto
  the agent core, so the editor binary is unaffected.
- Follow the layering proven in `forge-acp`: library-free transport port,
  capability negotiation, session store, protocol-to-runtime mapping, explicit
  projections, two-stage error mapping, contract tests with golden fixtures.

## Non-Goals

- No own wire types: unlike `forge` (which vendored protocol types over NDJSON),
  Zed already depends on the canonical `agent-client-protocol` crate (=2.2.0,
  features `unstable`, `unstable_protocol_v2`) and uses it for the client side.
  Reusing it in both directions avoids type drift.
- No HTTP/WebSocket transport in the first iteration (stdio only, as ACP
  prescribes for editor integration).
- No changes to how the Agent Panel consumes the agent in-process.

## Current Architecture (baseline)

- `crates/agent/src/agent.rs`: `NativeAgent` (GPUI entity, ~8k lines) owns
  `ThreadStore`, tools, permissions, models, skills.
- `crates/agent/src/native_agent_server.rs`: `NativeAgentServer` implements the
  internal `agent_servers::AgentServer` trait; `connect()` returns
  `Rc<dyn AgentConnection>`.
- `crates/acp_thread`: `AcpThread` entity + `AcpThreadEvent` — the internal
  session transcript with a clean event boundary, but no wire boundary.
- `crates/agent_servers/src/acp.rs` (5.6k lines): Zed's existing ACP **client**
  implementation (`Client` handlers, `Builder` dispatch, foreground work queue
  via `mpsc`). The mirror-image pattern for the server side already exists here.
- GPUI provides a headless platform (`crates/gpui/src/platform.rs`), enabling a
  GUI-free hosting process.

## Target Architecture

Layering adapted from `forge-acp`. Each layer is a module in the new crate
`acp_agent_server`; the agent core is only reached through L4.

```
stdio ndjson
     |
 [L7 Hosting binary "zed-acp"]
     |
     v
+------------------------------------------------------+
| L0 Transport port (stdio loop, memory pair for tests)|
| L1 Protocol / capability negotiation                 |
| L2 Session store (SessionId -> Thread/Connection)    |
| L3 Server / session mapping (impl ACP Agent trait)   |
| L5 Projections: permissions, tool calls, usage       |
| L6 Error mapping (anyhow -> ACP error codes)         |
+------------------------------------------------------+
     |  (L4 Deps port — the ONLY seam to the core)
     v
 NativeAgent / AcpThread / ThreadStore / Project
```

| Layer | forge-acp analogue | Zed implementation |
|---|---|---|
| L0 Transport | `transport.rs`, `stdio.rs` | `agent-client-protocol` stdio helpers + own memory transport pair for contract tests |
| L1 Protocol | `schema.rs`, capability negotiation | Types from `agent-client-protocol`; `initialize`/`authenticate` handlers, `AgentInfo`, session capabilities (load, close, modes, config options) |
| L2 Session store | `store.rs` (`SharedAcpSessionStore`) | `SessionId -> Entity<Thread>` + `WeakEntity<NativeAgent>` map; resume via existing `ThreadStore` |
| L3 Session mapping | `server.rs`, `session_handlers.rs` | `impl Agent`: `session/new`, `session/load`, `session/prompt`, `session/cancel`, `session/truncate`, `set_mode`, `set_config_option`, `session/list`; `AcpThreadEvent` -> `session/update` notifications |
| L4 Deps port | `deps.rs` (`AcpAgentServerDeps`) | Narrow struct/trait: `Arc<dyn Fs>`, `Entity<ThreadStore>`, project bootstrap, settings/credentials, `Templates` |
| L5 Projections | `policy_adapter`, `skill_approval`, `usage` | Tool permissions -> `session/request_permission`; elicitation; `AcpThread` entries -> `ToolCallUpdate` chunks; token usage |
| L6 Error mapping | `error_mapping.rs` (two-stage) | `anyhow` errors -> ACP `ErrorCode`, trimmed messages for the wire |
| L7 Hosting | `forge-cli` | New binary: headless GPUI app, workdir -> project bootstrap, logging to stderr only (stdout is the protocol channel) |

### The Send boundary (core design decision)

GPUI entities are `!Send` and live on the foreground thread; the ACP dispatch
loop is async. Zed already solved this for the client direction
(`agent_servers/src/acp.rs` enqueues `ForegroundWork` via `mpsc` onto the
foreground). The server side mirrors it:

1. The stdio read loop runs on the background executor and feeds an `mpsc`
   channel.
2. The `Agent` builder is constructed inside the GPUI app context; every request
   handler ships work onto the foreground (entity updates happen there) and
   answers flow back through responders/one-shots.
3. `NativeAgentConnection` is held as `Rc` on the foreground; the wire loop never
   touches it directly, only via the dispatch queue.

## Milestones

### M0 — Spike: headless GPUI + wire roundtrip

- Test binary answering only `initialize`; stdio NDJSON loop inside a headless
  app; proof of `AgentSideConnection` setup, foreground dispatch, clean shutdown
  on stdin EOF.
- Gate: roundtrip with a real ACP client.

### M1 — Crate skeleton, L0/L1

- New workspace crate `acp_agent_server` (`[lib] path = "src/acp_agent_server.rs"`,
  no `mod.rs`), per repo conventions.
- Transport port with memory pair; capability negotiation; `AgentInfo`.
- forge-parity contract tests: transport contract, capability negotiation,
  golden fixtures (request/response pairs as files) — first against a stub agent.

### M2 — L3/L2: first real session

- Define the deps port; bootstrap `NativeAgent` in the headless app (project on
  `session/new` workdirs, `ThreadStore` on a separate data dir, models and
  credential bootstrap without editor UI).
- `session/new` -> `session/prompt` -> streamed `session/update` from
  `AcpThreadEvent`s -> terminal `stopReason`.
- Tests: prompt-turn contract tests with `TestAppContext` plus stdio end-to-end.

### M3 — L5 projections

- Tool call rendering (incl. diffs/terminal cards), permission request
  projection (non-blocking; forge pattern: ask projection inside the tool-call
  flow), elicitation, cancel/abort propagation.

### M4 — L2 expansion: resume, listing, modes

- `session/load` via `NativeAgent::load_thread`, `session/list` via
  `ThreadStore` summaries, `set_mode`/`set_config_option` (model picker etc.),
  slash commands (`available_commands`), `truncate`/`retry`, title setting.

### M5 — Hardening

- Two-stage error mapping (L6), transport timeout/teardown behavior (aligned
  with the recent MCP HTTP teardown fixes), `wire_trace` debug log, exit codes,
  stdin-EOF behavior, `AcpThreadEvent` subscription leak checks.

### M6 — Integration and packaging

- Decision: standalone binary (`zed-acp`) vs. `zed` subcommand.
  Recommendation: standalone binary to avoid collisions with editor process
  specifics (e.g. `windows_subsystem`, startup paths); record as an ADR.
- Upstream strategy: crate + binary PR separate from layer work.
- Docs: configuring Zed's agent as an ACP server in other tools/editors.

## Risks

| Risk | Rating | Mitigation |
|---|---|---|
| NativeAgent bootstrap dependencies (settings store, release channel, credential store, language models assume an editor environment) | High (M2) | Deps port forces an explicit list; M0 spike proves it first; fall back to incremental feature gates |
| Send boundary: wire loop vs. GPUI foreground | Medium (M0) | Mirror the proven `ForegroundWork` pattern from `agent_servers/src/acp.rs` |
| Headless platform maturity on Windows | Medium | Run M0 on Windows first; gpui has headless abstractions but a windowless I/O loop is less exercised |
| Persistence collision (editor and server both writing ThreadStore SQLite) | Low | Default: separate data dir for the server; sharing is an explicit opt-in |
| Unstable protocol features (=2.2.0, `unstable_protocol_v2`) | Low | Keep everything behind capability negotiation; golden fixtures pin wire behavior |

## Open Questions

- Should the server share credential/model configuration with the desktop app
  (Zed config dir) or maintain its own isolated profile by default?
- Which ACP protocol version to negotiate as the baseline — v1 for maximum
  client compatibility, or v2 (`unstable_protocol_v2`) where available?
- Upstream appetite: land the crate without the binary first (pure library, no
  user-facing behavior), then the binary in a follow-up PR?
