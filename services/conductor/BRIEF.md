# Conductor — Agent Instructions

> "Conductor" is a working name. See Open Decisions.

## Mission

You are building Conductor: an engine that turns live input (data signals, events, plain-language requests and existing material) into music that plays without end and changes as it goes. Claude composes Strudel pattern code, and a real-time conductor performs it. One core engine runs many products, each a configured **station**: an IDE soundtrack, weather and place radio, chat-controlled radio, a shared never-ending song, adaptive scores for games and stories, focus and sleep radio, and a composing tutor and remix tool.

This is production-grade software: stable, tested, deployable. No hacks, no shortcuts. A running station must never go silent or stop because of a bad input, a model failure or invalid generated code.

## Hard Constraints

- Work only within `services/conductor/`. You may **read** `services/strudel/` as reference (docs, `doc.json`, examples) but never modify it or import from it. You may edit the root `deploy.yaml` to add Conductor's services.
- Do NOT use placeholder/stub implementations. Every public API must be fully functional.
- **Generated code is untrusted.** Model output must never run until the Validator (below) accepts it. That includes every client that plays it. No generated code may reach the network, storage, the DOM or globals except through the Strudel functions on the allowlist.
- **Never go silent.** Every failure path (model error, timeout, failed validation, crashed renderer) falls back to the last pattern that passed validation, then to the station's built-in safe pattern.
- No network calls in unit tests. Model calls in tests use recorded fixtures (record/replay). External APIs (weather, chat) are mocked behind their adapter interface.
- No secrets in code, fixtures, logs or the event log. API keys come only from environment variables.
- Strudel is AGPL-3.0. Every component that bundles or serves Strudel code must comply: license headers and a reachable source link from every served UI. Do not add dependencies whose licenses conflict with AGPL-3.0.
- Pin exact versions of every dependency, including `@strudel/*`.
- New runtime dependencies beyond the Tech Stack list need a one-line justification in `AGENTS.md`.

## Tech Stack

- TypeScript (strict, ESM) on Node 24 + pnpm workspaces for all packages
- `@strudel/core`, `@strudel/mini`, `@strudel/tonal`, `@strudel/transpiler`, `@strudel/webaudio` (pinned, from npm) as the music runtime
- Anthropic TypeScript SDK for the Composer and Interpreter: latest Claude models, configurable per role (a stronger model for composing, a fast model for interpreting and moderating)
- Zod for every schema at a boundary (protocol messages, station configs, model outputs)
- Fastify + `ws` for the HTTP/WebSocket API
- `@modelcontextprotocol/sdk` for the MCP server
- SQLite (`better-sqlite3`) for the event log and station storage, behind a repository interface (see Open Decisions)
- Playwright-driven headless Chromium + ffmpeg for broadcast rendering (HLS, plus optional Icecast)
- Vite + a small framework of your choice (React or Svelte; up for discussion) for the web player
- VS Code Extension API for the IDE extension
- Open-Meteo for weather (no API key)
- Vitest for unit and integration tests; Playwright for end-to-end tests
- Docker images deployed through the root `deploy.yaml`

## Architecture

### Package layout (`services/conductor/packages/`)

| Package | Role |
|---|---|
| `protocol` | Shared types and Zod schemas: inputs, intent, state, log events, WebSocket messages, station config. No runtime dependencies besides Zod. |
| `state` | Musical State model, the append-only event log, a repository interface and a SQLite implementation. |
| `interpreter` | Fast mappings (signal → intent/control), model-based request interpretation, the crowd layer. |
| `composer` | Prompting, context assembly (state, intent, Strudel reference, style profile), generating ahead, edits to one layer at a time, explanations. |
| `validator` | Code check against an allowlist, sandboxed dry run, musical checks. Runs on the server and the client. |
| `runtime` | Browser-side Conductor: Strudel integration, the `ctl()` control signal, applying code on bar lines, crossfades, change limits. |
| `adapters/*` | One package per input source: `text`, `weather`, `ide-events`, `chat-twitch`, `game-events`, `webhook`. |
| `server` | Station manager, HTTP/WebSocket API, composition scheduler, MCP server. |
| `broadcaster` | Headless render of a station to HLS/Icecast. |
| `player` | Web app: station player, code view, request box, history and fork. |
| `vscode` | IDE extension: webview player + editor, test and debug event adapter. |
| `cli` | Admin and dev tool: create/list stations, send inputs, tail the log, render to file. |

### Key structural decisions

- **The server owns the state; clients render audio.** The server owns the Musical State, the event log and composition. Clients (player, IDE extension, broadcaster) receive validated code and control values, then play them locally. One composition serves any number of listeners.
- **Event sourcing.** Every change to a station (input received, intent changed, code proposed, accepted, rejected, applied) is an immutable log event with author, reason and timestamp. Current state can be rebuilt from the log. History, fork, replay and share links all come from the log.
- **Control at three speeds.**
  - *Fast* (ms to seconds, no model): signals move named controls continuously.
  - *Medium* (bars): the Conductor switches between variations and cues generated in advance.
  - *Slow* (phrases to minutes): the Composer rewrites code.
  - The model is never in the real-time path.
- **The pattern contract.** All Composer output follows one contract, enforced by the Validator:
  - Code is split into named layers (`drums`, `bass`, `harmony`, `melody`, `texture`, plus custom layers) using Strudel's labeled-pattern syntax. Verify against the pinned Strudel version.
  - Continuous parameters read engine controls through `ctl(name)`, a Strudel signal the runtime registers. The core controls are `energy`, `tension`, `brightness`, `density` and `space`; station-defined controls are allowed.
  - Edits may target a single layer.
- **A station is a configuration, not code.** It is a validated document: adapters + mappings + style profile + policies (change limit, how often the model composes, moderation) + form plan + outputs. Adding a product means writing an adapter and/or an interface, never forking the engine.
- **Dependency injection throughout.** No module-level mutable state. Clock, random seed, model client and storage are all injected so tests are deterministic.

## Feature Requirements

### Input Adapters

- A common adapter interface that emits four kinds of input: **Signal** (named continuous value), **Event** (named discrete occurrence with a payload), **Request** (plain-language text with an author), **Material** (Strudel code, prose text, or a share link to remix).
- `text`: requests from the API, player, MCP and CLI.
- `weather`: Open-Meteo conditions, time of day, sunrise/sunset and season for a configured place. Polled on a configurable interval.
- `ide-events`: file edits, typing rate, diagnostics count, test start/pass/fail, debug session start/stop, run/deploy tasks. Sent by the VS Code extension.
- `chat-twitch`: reads chat from a configured channel and turns `!` commands and free text into Requests.
- `game-events`: an authenticated WebSocket/HTTP API that games and interactive-fiction runtimes use to send Events and Signals, plus a small TypeScript client SDK.
- `webhook`: a generic authenticated HTTP endpoint that accepts any input kind. It is the integration path for wearables and other sources before dedicated adapters exist.

### Interpreter & Crowd Layer

- Fast mappings defined in station config: signal → control, with range, curve, smoothing and dead zone. Pure functions, unit-tested.
- Event mappings: event → control change, cue switch, or "compose now" trigger.
- Request interpretation through the model: free text → a structured **intent change** (Zod-validated), with a short human-readable summary. Text that has no musical meaning is rejected with a reason.
- Crowd layer for many-to-one stations:
  - Vote rounds with a configurable window and quorum
  - Per-user and global rate limits
  - Model-assisted moderation plus a configurable blocklist
  - A cap on how far one round can move the intent
  - Credit recorded in the log

### Musical State & Session Log

- **Intent:** energy, valence, tension, density, brightness, tempo (BPM), key and mode, sound palette, genre/style tags.
- **Form plan:** an ordered list of sections with target intent and duration, or a cue graph for event-driven stations. Built-in templates:
  - Endless evolving (radio)
  - Long fade to silence (sleep)
  - Work/break cycles (focus)
  - Cue graph (games and stories)
- **Theme library:** named motifs (melodies, rhythms, progressions) attached to a station, character or place. The Composer reuses and develops them.
- **Current state:** the active code, active layer set, control values and the musical time position (cycle).
- Event log API: append, query by station and time range, rebuild state at any point, **fork** a station from any event, and **share link** for a snapshot.

### Composer

- Builds model context from:
  - Current state and target intent
  - Form section
  - Themes
  - Style profile
  - Recent history
  - A curated, relevant slice of the Strudel function reference generated from `doc.json`
- Produces a full pattern or an edit to one layer. Output always follows the pattern contract.
- **Generates ahead:** keeps a buffer of validated next variations and, for cue-graph stations, a validated cue bank, so the Conductor never waits on the model.
- On a validation failure, retries with the Validator's errors in the prompt, up to a configured limit, then falls back. Never ships unvalidated code.
- Produces an optional **explanation** (what changed and why, in musical terms) for tutor and remix interfaces.
- Enforces a per-station budget on model calls, with metrics.
- Supports multiple **style profiles** (palette, tempo range, allowed instruments and samples, harmonic language, reference examples), editable as data.

### Validator & Sandbox

- **Code check:** parse with Strudel's transpiler/acorn and reject anything outside the allowlist.
  - Identifiers are limited to Strudel's exported functions, `ctl`, local arrow-function parameters and literals.
  - No access to `window`, `globalThis`, `fetch`, `import`, `eval`, `Function`, `constructor`, `__proto__`, `prototype` or computed member access to such names.
  - Sample sources come only from the station's allowlist.
- **Dry run:** evaluate in an isolated context with no audio and query the pattern over N cycles. Reject it if it:
  - Throws
  - Times out
  - Produces no events in a layer marked active
  - Exceeds a maximum event density
  - Breaks the pattern contract
- **Musical checks:** tempo and key consistent with the intent unless the change is deliberate; layer set matches the declared layers.
- The same Validator package runs on the client before evaluation, as a second line of defense.
- Returns structured errors that the Composer can use.

### Conductor (Runtime)

- Plays Strudel through `@strudel/webaudio`. Registers the `ctl()` signal so controls update between cycles with no re-evaluation.
- Applies new code only on bar or phrase lines (configurable: 1/2/4/8 cycles). Crossfades or brings in layers one at a time when the layer set changes.
- Enforces each station's change limit, the largest intent change per phrase, so the music evolves instead of lurching.
- Cue switching for event-driven stations: switches to the target cue on the next beat or bar.
- Tracks musical time and reports position, playing state and errors back to the server.
- Recovers from audio-context interruption, tab sleep and reconnection without user action beyond the browser's required first click.

### Renderers & Outputs

- **Local:** the player, IDE webview and embeds render in the listener's browser.
- **Broadcast:** the `broadcaster` runs a station in headless Chromium, captures PCM, encodes it with ffmpeg to HLS (and Icecast when configured), and normalizes loudness. It is supervised and restarts on crash, carrying on from the current state.
- **Export:**
  - The current code (with explanation)
  - A share link (a snapshot in the log)
  - A WAV render of N minutes, used by the audiobook/podcast pipeline

### Product Interfaces

- **Web player:**
  - Station directory
  - Player with now-playing intent and the current section
  - Code view with explanations
  - Request box
  - History timeline with replay and fork
  - Share links
- **VS Code extension:**
  - Webview player
  - Editor, test and debug event adapter
  - Status-bar controls (play, pause, intensity)
  - A "do not disturb" policy that keeps changes gentle while you type
- **MCP server:** `list_stations`, `create_station`, `get_state`, `send_input`, `set_intent`, `get_code`, `propose_code`, `explain`, `fork`, `export_render`. It is the main interface for the tutor, "start from a feeling" and remix.
- **Chat radio:** the Twitch adapter plus an on-stream overlay page showing the current vote, the last accepted change and credits.
- **Game and story API:** the event SDK, cue-graph station templates, and a sample interactive-fiction page that exercises them.
- **Tabletop RPG console:** a web page with a scene text box, scene presets and themes per scene, built on the same API.
- **Audiobook/podcast pipeline:** text with chapter and scene markers → form plan → cue bank → rendered audio aligned to the markers. Runs from the CLI and the API.

### Stations (shipped templates)

Each template is config only:

- `weather-radio`
- `focus`
- `sleep`
- `ide`
- `chat-radio`
- `shared-song`
- `game-score`
- `tabletop`
- `audiobook`
- `sketchpad` (used by tutor, feeling start and remix)

### Observability

- Structured logs with station ID, and metrics:
  - Model calls, latency and cost per station
  - Validation pass rate
  - Fallback count
  - Silence detection (broadcast)
  - Connected listeners
- A health endpoint per service.

## Out of Scope (v1)

- **Bandmate.** It needs real-time listening to live audio or MIDI. Design the input interface so it can be added later.
- Dedicated smart-watch and wearable integrations. Wearables can send data through the generic `webhook` adapter.
- Neural audio generation, vocals or realistic instrument rendering.
- Native mobile apps; Unity, Godot or Unreal plugins (the TypeScript SDK and HTTP API only).
- MIDI file and stem export.
- Voice input (tabletop and other interfaces take text only).
- Billing, payments, full user accounts (v1 uses station owner tokens + anonymous listeners), multi-region scaling.
- Chat platforms other than Twitch.

## Definition of Done

- All features listed above implemented and passing unit tests.
- **Soak:** a broadcast station runs 24 hours with zero silent gaps over 500 ms, no unhandled errors, and memory growth under 10%.
- **Latency:**
  - A signal change becomes audible within 250 ms on local playback.
  - A game cue switch lands on the next beat (≤ 500 ms at 120 BPM).
  - Requests take effect at the next phrase boundary after composition, typically within 30 s.
- **Composer quality:** at least 95% of generated patterns pass validation on the first attempt across the evaluation suite (Testing below). Fallback works 100% of the time.
- **Loudness (broadcast):** −16 LUFS ±2 integrated, true peak ≤ −1 dBTP.
- **Cost:** with default settings, each station stays within its configured model-call budget (default ≤ 60 calls per station-hour).
- **Security:** the Validator rejects 100% of the adversarial corpus (escape attempts, network access, prototype tampering, resource exhaustion).
- **Build:** zero type errors and lint warnings. Unit tests run in under 60 s with no network access.
- **Deploy:** `docker compose -f deploy.yaml` (or the stack equivalent) brings up the server, broadcaster and player, which work end to end. The VS Code extension packages to a `.vsix` that installs and plays.

## Testing

- Write unit tests for every public API function. Framework: Vitest.
- **Deterministic music tests:** query patterns over fixed cycle ranges with an injected seed and clock, then assert on the resulting events. Use snapshots for golden patterns.
- **Model fixtures:** Composer and Interpreter tests replay recorded model responses. A separate, opt-in `pnpm test:live` suite hits the real API.
- **Validator corpus:** a growing set of valid patterns that must pass and adversarial patterns that must fail.
- **Integration station:** a demo station that exercises every adapter, every input kind, every control speed, fork and replay, export, and every MCP tool.
- **Evaluation suite:** a fixed set of prompts and inputs per station template, run against the live model. It measures:
  - Validation pass rate
  - Intent adherence (tempo, density and key computed from the queried events)
  - Change limit compliance
  - Results tracked over time in `eval/results/`
- **End to end (Playwright):** player playback, request → audible change, code view, fork, IDE webview, broadcaster output stream.
- Run the full test suite before every commit. Do not commit if tests fail.

## Code Style

- TypeScript `strict`, ESM, Prettier + ESLint (typescript-eslint, recommended-type-checked).
- Naming:
  - Files: `kebab-case.ts`
  - Types and classes: `PascalCase`
  - Functions and variables: `camelCase`
  - Constants: `UPPER_SNAKE_CASE`
- Named exports only. One public entry point (`src/index.ts`) per package. Keep internals unexported.
- Validate with Zod at every boundary (network, storage, model output). Inside those boundaries, trust the types.
- Return typed results for expected failures (validation, model refusal, rate limit). Throw only for programmer errors.
- No module-level mutable state. Pass dependencies explicitly.
- Keep interpreter mappings and validator rules as pure functions.
- Every source file distributed to clients carries the AGPL-3.0 header.

## Git Workflow

- Repository: https://github.com/tblease/strudel (Conductor lives in `services/conductor/`).
- Before every commit: self-review for style compliance, correctness, and test coverage.
- Commit messages: Conventional Commits with the package as scope, e.g. `feat(composer): generate cue banks ahead of playback`, `fix(validator): reject computed constructor access`.
- Push after every logical unit of work.

## Workflow Rules

- Work autonomously. Ask questions only when a decision is genuinely blocking and cannot be reasonably inferred.
- When starting:
  - Confirm the project name (suggest 3 options).
  - Confirm or override each Open Decision below, using the stated recommendation as the default if there's no answer.
- Create and maintain `services/conductor/AGENTS.md` with concrete development guidelines: commands, package map, the pattern contract, how to record fixtures, how to add an adapter or a station template.
- Prioritize: correctness > test coverage > performance > polish. Security and "never silent" count as correctness.

## Milestones

| Phase | Deliverable | "Done" means |
| ----- | ----------- | ------------ |
| 1 | Workspace, `protocol`, `state`, `validator`, `runtime`, `cli`, CI | A hand-written pattern following the contract plays in a browser, `ctl()` controls move it live, every state change is logged and replayable, the adversarial corpus is rejected, CI green |
| 2 | `composer`, `interpreter` (requests), `text` adapter, `server`, MCP server, web player with code view, `sketchpad` template | Plain-language requests produce validated, explained patterns that change at phrase boundaries; fork and share work; the tutor/feeling/remix flows work through both the player and MCP; the evaluation suite has a baseline |
| 3 | Fast mappings, form plans, theme library, `weather` + `ide-events` + `webhook` adapters, VS Code extension, `weather-radio`/`focus`/`sleep`/`ide` templates | Each template runs for 8 hours locally with coherent sections; a signal reaches the audio within 250 ms; the `.vsix` installs and reacts to tests passing and failing |
| 4 | `broadcaster`, crowd layer, `chat-twitch` adapter, overlay, `chat-radio`/`shared-song` templates | The 24-hour broadcast soak passes; chat votes change the stream with credit; moderation and rate limits are tested; the shared song's history can be scrubbed and forked |
| 5 | Cue banks, `game-events` SDK, tabletop console, audiobook pipeline, `game-score`/`tabletop`/`audiobook` templates, docs | Cue switches land on the next beat; the sample interactive-fiction page and tabletop console work end to end; a chapter renders to aligned audio; every package has usage docs; every Definition of Done item passes |

## Open Decisions

- **Project name.** Recommendation: choose at kickoff. "Conductor" is only a working name and is generic.
- **Strudel source: npm vs. the vendored copy.** Recommendation: pinned `@strudel/*` packages from npm. That keeps Conductor independent of the vendored REPL in `services/strudel/` and makes upgrades deliberate. Use the vendored copy only as read-only reference.
- **Raw Strudel code vs. a constrained format that compiles to Strudel.** Recommendation: raw Strudel restricted by the AST allowlist and pattern contract. Readable, editable code is what makes the tutor, remix, share and fork worth having. Revisit if the evaluation suite's first-try validation rate stays under 95% after prompt work.
- **Broadcast rendering: headless Chromium vs. Node-native (`supradough`).** Recommendation: headless Chromium. It is the reference audio engine, so broadcast sounds exactly like local playback. `supradough` is a proof of concept; reassess for faster-than-real-time offline rendering (audiobooks) later.
- **Storage: SQLite vs. Postgres.** Recommendation: SQLite behind the repository interface for v1 (single node, simple backups). The interface makes switching to Postgres a contained change when multi-node is needed.
- **Licensing posture.** Recommendation: release Conductor under AGPL-3.0. Anything that bundles Strudel (player, VS Code extension, broadcaster) must comply anyway, and a single license avoids ambiguity. Confirm before any commercial plans.
- **Sample library.** Recommendation: self-host a curated, license-checked sample set, with a per-station allowlist, rather than pulling from third-party repositories at runtime. That keeps it reliable, auditable and safe to broadcast.
- **First chat platform.** Recommendation: Twitch (its chat API is simple and live-coding streams are common there), with the generic `webhook` adapter for everything else until demand is clear.
- **Player UI framework.** Recommendation: React, for its ecosystem and familiarity to contributors; Svelte is fine if bundle size for embeds becomes a priority.
