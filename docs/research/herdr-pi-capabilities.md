# Herdr and pi: Starbridge MVP capability research

Research for [Verify Herdr and pi capabilities required by the MVP](https://github.com/smotastic/starbridge/issues/2), within [Starbridge v0.1 — MVP scope and feasibility map](https://github.com/smotastic/starbridge/issues/1).

## Finding

**A plausible integration exists, but Herdr status alone cannot implement the unattended promise.** Herdr provides persistent terminals, interactive launch/input, runtime snapshots, and native pi conversation references. Pi provides lifecycle events and extension tools suitable for explicit mission reports. Neither supplies Starbridge's mission success semantics, repository-wide execution ownership, durable delivery acknowledgements, or a confirmed whole-mission termination primitive. These remain design choices, not facts established by this research. [H1–H6, P1–P5]

The least disruptive candidate under the fixed **Herdr → pi** requirement is interactive pi in a Herdr shell pane, plus a small explicit mission-reporting/control extension and an independently observable process-lifetime boundary. A persistent runner around pi RPC is an alternative with cleaner machine I/O but more infrastructure and a different human inspection experience. Neither option is selected here.

## Evidence and version boundary

Research performed directly with user approval on 2026-09-05; no research subagent was available. This is documentation/source investigation with a limited no-model RPC smoke check, **not an end-to-end feasibility certification**.

Observed locally:

| Component | Observation |
| --- | --- |
| Host | `uname -m`: `aarch64` |
| Herdr client | `herdr --version`: `0.8.2` |
| Herdr server | `herdr status server`: running, `0.8.2`, protocol `20`, compatible |
| Pi | `pi --version`: `0.85.1`; npm package `@earendil-works/pi-coding-agent` |
| Node | `v22.23.2`; pi package requires `>=22.19.0` |
| Herdr pi integration | `herdr integration status`: **outdated (`v6 < v8`)** |
| Installed integration | `~/.pi/agent/extensions/herdr-agent-state.ts`, SHA-256 `12efd27592fbf185343f035ee4fd2b1999e9356e323e4b01225f7d6bc89b16de` |
| Published Herdr artifact | Release `v0.8.2` includes `herdr-linux-aarch64` |

Herdr source inspected at commit `9eb521456ac0d19d3ab3d9d7cea3cca10baa8a4c` (peeled `v0.8.2` tag). Pi installed docs/JavaScript were inspected for `0.85.1`; its release tag resolves to `d981de1229ef899957bbe968bc8dcda02a21f477`. Links below pin these revisions. Herdr's checked-in stable documentation is under `docs/versions/0.8.0/` in the `0.8.2` source tree; installed CLI/schema and release source take precedence over examples. [H0, P0]

No integration was installed or updated, no existing agent was prompted/stopped, no live server was restarted, and no repository credentials or conversation contents were published.

## Capability matrix

“Supported” means documented/source-backed, not that Starbridge already implements it.

| Need | Supported interface / evidence | Limit or implication |
| --- | --- | --- |
| Mission workspace | `herdr worktree create --cwd PATH --branch NAME --base REF --path PATH --no-focus`; `worktree list/open`; workspace worktree provenance | Returns actual workspace/worktree identities; persist responses rather than manufacture IDs. Git worktree isolation is not a permission boundary. [H1] |
| Terminal identity | `pane get/list`, `agent get/list`, `herdr api snapshot`; workspace/tab/pane/terminal IDs; `cwd`, `foreground_cwd`, optional `agent_session` | Pane IDs change on cross-workspace moves; agent names belong to live occupants, not permanent missions. Correlate against a Starbridge-owned identity and current server context. [H1, H2, H3] |
| Launch pi | `herdr agent start NAME --kind pi --pane ID --timeout MS -- <pi-args>` | Requires an existing available shell pane. Does not create a workspace. Default timeout 30 seconds; source permits >3 and <=300 seconds. Startup readiness is not successful mission execution. [H2, H3] |
| Prompt submission | `herdr agent prompt TARGET TEXT --wait --timeout MS`; socket `agent.prompt` with optional `wait` | Text/Enter submission is atomic and bracketed-paste aware. Already-blocked agents reject prompt submission before input. Wait pins occupant; does not identify a specific mission attempt. Sending while already working can settle on the earlier work. No durable exactly-once prompt acknowledgement. [H1, H2, H4] |
| Progress | Herdr semantic state and `state_change_seq`; `events.subscribe`; pi `agent_start`, tool/turn/message events | State snapshots are not a durable event journal. Terminal output is presentation, not a reliable protocol. Pi structured progress needs an extension bridge, JSON stream, or RPC client. [H1, H3, H5, P1, P2] |
| Ready for verification | Pi `agent_settled`; custom tool results with structured `details` | Settled means no automatic continuation remains, not “requirements met.” Require an explicit mission/attempt report plus independent checks. `agent_end` is weaker: retry, compaction, or follow-up can still occur. [P1, P2, P5] |
| Blockers | Pi `ui_prompt_start/end`; RPC dialog `extension_ui_request`; custom reporting tool | Natural-language uncertainty is not automatically a machine blocker. Herdr's bundled pi integration listens for `herdr:blocked`, not pi's native `ui_prompt_start/end`: an explicit bridge/report is needed for that path. [H5, P1, P2, P6] |
| Failure | Pi assistant `stopReason` (`error`, `aborted`, etc.), tool `isError`, retry/compaction events; process exit | Tool errors can be recoverable; low-level idle or process exit cannot alone classify mission outcome. Missing reports must remain ambiguous rather than succeed. [P1, P2, P3] |
| Stop execution, retain pane | Herdr `agent send-keys`; pi abort, `/quit`, extension `ctx.shutdown()`; OS process observation | **No `agent stop/kill` API** in this Herdr CLI/schema. Keys request UI behavior, not a confirmed process-tree stop. `pane.release_agent` changes reporting authority, not process lifetime. Retain the shell pane instead of closing it. [H1, H2, H5, P1, P2] |
| Observe process identity | `herdr pane process-info --pane ID`: shell PID, foreground process group, foreground processes | Supported and observed on this ARM64 host. Not a complete descendant inventory, PID-generation identifier, or atomic stop-and-confirm API. Foreground shell does not prove no detached child survives. [H3, P7] |
| Retained inspection | Worktree filesystem, shell pane, `pane read`, terminal attach; saved pi JSONL | Agent-specific targeting may vanish after exit; use retained pane/terminal identity. Alternate-screen output is not a complete transcript. Pi session file is the stronger conversation artifact. [H1, H2, H6, P3] |
| Starbridge-only crash | Herdr server owns terminals independently; reconnect and query snapshots | Can resume observing a still-live pi without launching another. Must reconcile identities and unresolved sends; no automatic Starbridge mission adoption protocol is supplied. [H1, H6] |
| Herdr restart / reboot | Layout restore, optional screen history, native `pi --session` restore | **Original processes do not survive.** Restore can start new pi processes, is enabled by default, and needs client sizing/theme context. This is not live supervision reattachment. [H6, H7] |
| ARM support | Herdr Linux aarch64 release; installed pi/Node run on aarch64 | Evidence for this 64-bit stack, not 32-bit Raspberry Pi OS, minimum RAM, thermal limits, repository builds, or provider reliability. [H0, P0; local observations] |

## Important integration details

### 1. Do not equate Herdr “done” with mission completion

Herdr `done` is idle background work that has not yet been seen in the UI; focusing the tab changes the presentation. `blocked` is recognized waiting-for-input state, and `unknown` is lack of classification. A prompt wait normally settles on `idle`, `done`, or `blocked`. The prompt-stall check detects absence of an observed lifecycle transition within five seconds when starting from a non-working state; it is not an application-level receipt or proof that no work occurred. [H2, H4]

Pi's `agent_settled` is the appropriate lifecycle boundary, but can follow failure or abort as well as a useful answer. A custom final-report tool is supported, and its `details` can carry structured data. Its `terminate: true` only stops automatic follow-up when **every finalized tool result in the batch** is terminating; it does not kill the process or forbid queued/new input. [P1, P2, P5]

**Implication:** distinguish an agent report, settled execution, supervisor verification, and completed mission. They are separate evidence, not interchangeable statuses.

### 2. Bundled integration is a best-effort UI integration, not mission transport

Herdr `0.8.2` bundles integration version 8. The installed version 6 differs materially:

- Version 8 gates on `ctx.mode === "tui"`; version 6 gates on `ctx.hasUI`, which is also true in RPC mode.
- Both report native session references, working on `agent_start`, and idle on `agent_settled` when `ctx.isIdle()` is true.
- Both listen for the custom event bus topic `herdr:blocked`; neither inspected asset subscribes to native pi `ui_prompt_start/end`. Pi's native runner emits those lifecycle events separately. A standard dialog-to-Herdr-blocked mapping is therefore not established by installing this asset alone.
- Version 6 sends `pane.release_agent` on `session_shutdown` with reason `quit`; version 8 no longer does so. Do not derive process-death guarantees from this hook.
- Sending uses a 500 ms attempt followed by a 1500 ms retry. Receiving any socket data counts as delivery; the response body is not checked for success. A queued state may be replaced by a newer state while a send is in progress; unchanged states are suppressed. There is no persistent outbox or retry-until-acknowledged contract. [H5; locally inspected v6 file]

**Implication:** pin and verify the integration version. A future extension should live beside, not edit, Herdr's managed file. Missing/stale telemetry must not authorize mission handoff or replacement execution.

### 3. Confirmed termination is an actual design gap

Pi supports cancellation and graceful shutdown, but these have different meanings:

- Interactive Escape aborts; Ctrl+C clears the editor, and Ctrl+C twice quits. Configurable keybindings and current UI state make blind key sequences unsuitable as the sole safety proof. [P0]
- RPC `abort` waits for idle, but queued messages can continue unless `clear_queue` runs first. Neither means process exit. [P1]
- Extension `ctx.shutdown()` is deferred to an idle point in interactive/RPC modes; it is not a hard timeout enforcement operation. Session shutdown also happens on reload/new/resume/fork, not only process quit. [P2]
- Pi tracks built-in detached child processes for shutdown cleanup and uses process-group kills on Unix. This is useful implementation evidence, not a guarantee for arbitrary extensions, escaped descendants, abrupt SIGKILL, or independent processes started outside the tracked lifecycle. [P7]

Herdr's process-info can anchor observation, but an available shell / missing agent label does not prove all mission-owned processes stopped. An independently managed process boundary with identity-safe stop confirmation is a candidate; its scope and failure behavior must be decided. **If stopping cannot be established, halt the run rather than starting another mission**, as already required by the map baseline.

### 4. Recovery has three distinct cases

1. **Starbridge dies, Herdr and pi survive:** reconnect to the same Herdr server and correlate live pane/process/native session with the persisted mission. Do not replay an uncertain prompt blindly. Herdr snapshot plus current state is available; no durable event replay or application idempotency guarantee is documented. [H1, H4, H6]
2. **Pi exits, Herdr survives:** keep worktree and shell pane; retain the pi session path independently because the live agent identity can disappear. `pi --session <exact-path>` resumes conversation in a new process, not an interrupted OS operation. [H2, P0, P3]
3. **Herdr server stops or host reboots:** old processes are gone; restored layout and a newly resumed pi are different from a live process. Native restore defaults on (`[session].resume_agents_on_restore = true`), can activate eligible restored panes across tabs, and reconstructs pi as `pi --session <reference>`, not the original full launch argument list. Exact Starbridge extension/flags cannot be assumed to survive that reconstruction. [H6, H7]

**Implication:** the ownership/restart ticket must decide whether to disable Herdr native auto-resume for the managed runtime, or otherwise reconcile/gate it before execution. This research makes no shared-config changes. Screen-history persistence is separately off by default and may contain secrets; enabling it is not necessary to retain the Git worktree or pi JSONL. [H6]

## Minimal feasible options — not selected

### A. Interactive pi + explicit mission extension

- Herdr creates/opens the mission worktree, starts interactive pi in an available shell pane, and exposes live UI/status.
- A pinned extension produces attempt-correlated reports such as needs-human / ready-for-verification, observes settled/error/UI lifecycle, and provides a control path whose acknowledgements and recovery semantics are explicit.
- Starbridge owns verification/handoff and durable mission correlation; process-lifetime confirmation remains separate from reports.

**Advantages:** closest to the fixed topology; human can inspect the actual TUI; supervisor can die without owning pi's stdin.

**Costs / unknowns:** durable reporting/command receipt, session replacement handling, blocker bridge, stop proof, and human interaction rules need design. Merely appending custom pi entries does not establish an fsync-backed mission journal. [H1–H6, P2, P3, P5]

### B. Persistent Herdr-hosted runner + pi RPC

- A small long-lived runner in a Herdr pane owns pi's stdin/stdout and durable machine events; Starbridge reconnects to the runner rather than spawning pi as its own disposable child.
- Pi RPC supplies correlated command acceptance, structured events, session IDs, state, queues, and dialog requests. Split framing on LF only.
- Retained Herdr access shows runner logs, not pi's interactive TUI. Native Herdr pi integration v8 deliberately excludes RPC, so status/inspection would need explicit integration. Opening an interactive session concurrently is not a safe substitute for inspection.

**Advantages:** cleaner bidirectional machine I/O and explicit queue/state queries.

**Costs / unknowns:** new runner/transport/lifetime ownership, stream persistence/backpressure, reconnect semantics, and stop proof. A bare Starbridge-spawned `pi --mode rpc` does not preserve the same crash-survival architecture: pi shuts down on stdin EOF. [H5, P1, P4]

### C. Herdr-hosted one-shot JSON invocation per attempt

`pi --mode json` / print mode can produce structured output, retain a pi session, and exit when prompts finish. A runner or shell can retain logs and exit status while leaving the pane available. [P0, P8]

**Trade-off:** simpler bounded attempt boundary, weaker live intervention; UI methods are no-ops in JSON/print, so an explicit blocker report is mandatory. Persistence, descendants, exit observation, and attempt correlation still need design. This does not make an exit code proof of requirements being met. The UI and resumption trade-off must be acceptable to the human.

**Not credible:** terminal scraping + `done` + Ctrl+C as the entire contract; direct RPC with no surviving owner while promising uninterrupted crash recovery.

## Smoke check and unverified work

A temporary isolated pi configuration and temporary cwd were used with `PI_OFFLINE=1`, no session persistence, no extensions/skills/context files/themes, and no approved project resources. No model prompt was sent.

Observed on this aarch64 host:

- `get_state`: successful correlated response, session ID, `isStreaming=false`, `isCompacting=false`, `pendingMessageCount=0`.
- `clear_queue`: successful response with empty steering/follow-up queues.
- `abort`: successful response while already idle.
- Closing stdin: process exited with code 0 within the ten-second wait.
- `herdr pane process-info --current`: returned shell PID, foreground process-group ID, and the current pi PID/name/cwd. This was read-only observation, not a termination test.

**Not tested:** agent launch/readiness in a new mission worktree, actual provider turns, extension report delivery, UI blocker bridge, cancellation while tools run, descendant cleanup, supervisor kill/reconnect during execution, Herdr cold restore, duplicate prompts, power loss, PR handoff, performance under Raspberry Pi build workloads, or 32-bit hosts. No production adapter was built.

These are candidate acceptance probes for the chosen contract, not additional implementation tickets. Research establishes supported primitives and gaps; it does not settle the human trade-offs in the supervision/recovery decisions.

## Primary sources

All code/document links below are pinned. Installed-command observations are listed above so they can be repeated; local docs root was the installed `@earendil-works/pi-coding-agent` package.

- **H0:** [Herdr v0.8.2 release / ARM64 assets](https://github.com/herdrdev/herdr/releases/tag/v0.8.2).
- **H1:** [Herdr socket API, CLI, worktrees, subscriptions, snapshots and schema](https://github.com/herdrdev/herdr/blob/9eb521456ac0d19d3ab3d9d7cea3cca10baa8a4c/docs/versions/0.8.0/website/src/content/docs/socket-api.mdx).
- **H2:** [Release agent skill and lifecycle semantics](https://github.com/herdrdev/herdr/blob/9eb521456ac0d19d3ab3d9d7cea3cca10baa8a4c/skills/herdr/SKILL.md); installed `herdr --help`, `herdr agent`, `herdr pane`, `herdr worktree`, `herdr session`, `herdr api schema`; [release agent launch implementation](https://github.com/herdrdev/herdr/blob/9eb521456ac0d19d3ab3d9d7cea3cca10baa8a4c/src/app/agents.rs), [prompt implementation](https://github.com/herdrdev/herdr/blob/9eb521456ac0d19d3ab3d9d7cea3cca10baa8a4c/src/app/api/agents.rs).
- **H3:** [Agent schema](https://github.com/herdrdev/herdr/blob/9eb521456ac0d19d3ab3d9d7cea3cca10baa8a4c/src/api/schema/agents.rs), [pane/process schema](https://github.com/herdrdev/herdr/blob/9eb521456ac0d19d3ab3d9d7cea3cca10baa8a4c/src/api/schema/panes.rs).
- **H4:** [Herdr wait and prompt lifecycle implementation](https://github.com/herdrdev/herdr/blob/9eb521456ac0d19d3ab3d9d7cea3cca10baa8a4c/src/api/wait.rs).
- **H5:** [Bundled pi integration v8](https://github.com/herdrdev/herdr/blob/9eb521456ac0d19d3ab3d9d7cea3cca10baa8a4c/src/integration/assets/pi/herdr-agent-state.ts).
- **H6:** [What survives detach/restart, native restore, screen history](https://github.com/herdrdev/herdr/blob/9eb521456ac0d19d3ab3d9d7cea3cca10baa8a4c/docs/versions/0.8.0/website/src/content/docs/session-state.mdx), [persistence and terminal attach](https://github.com/herdrdev/herdr/blob/9eb521456ac0d19d3ab3d9d7cea3cca10baa8a4c/docs/versions/0.8.0/website/src/content/docs/persistence-remote.mdx).
- **H7:** [Native resume command construction](https://github.com/herdrdev/herdr/blob/9eb521456ac0d19d3ab3d9d7cea3cca10baa8a4c/src/agent_resume.rs), [deferred restore execution](https://github.com/herdrdev/herdr/blob/9eb521456ac0d19d3ab3d9d7cea3cca10baa8a4c/src/app/agent_resume.rs).
- **P0:** [Pi package/version/Node engine](https://github.com/earendil-works/pi/blob/d981de1229ef899957bbe968bc8dcda02a21f477/packages/coding-agent/package.json), [CLI, modes, trust, session flags, keys](https://github.com/earendil-works/pi/blob/d981de1229ef899957bbe968bc8dcda02a21f477/packages/coding-agent/README.md).
- **P1:** [Pi RPC protocol](https://github.com/earendil-works/pi/blob/d981de1229ef899957bbe968bc8dcda02a21f477/packages/coding-agent/docs/rpc.md).
- **P2:** [Pi extension lifecycle, UI events, shutdown, tools and persistence](https://github.com/earendil-works/pi/blob/d981de1229ef899957bbe968bc8dcda02a21f477/packages/coding-agent/docs/extensions.md).
- **P3:** [Pi session format and SessionManager](https://github.com/earendil-works/pi/blob/d981de1229ef899957bbe968bc8dcda02a21f477/packages/coding-agent/docs/session-format.md).
- **P4:** [RPC runtime including stdin EOF shutdown](https://github.com/earendil-works/pi/blob/d981de1229ef899957bbe968bc8dcda02a21f477/packages/coding-agent/src/modes/rpc/rpc-mode.ts).
- **P5:** [Structured-output tool example](https://github.com/earendil-works/pi/blob/d981de1229ef899957bbe968bc8dcda02a21f477/packages/coding-agent/examples/extensions/structured-output.ts).
- **P6:** [Extension runner UI lifecycle emission](https://github.com/earendil-works/pi/blob/d981de1229ef899957bbe968bc8dcda02a21f477/packages/coding-agent/src/core/extensions/runner.ts).
- **P7:** [Pi shell process tracking/termination](https://github.com/earendil-works/pi/blob/d981de1229ef899957bbe968bc8dcda02a21f477/packages/coding-agent/src/utils/shell.ts).
- **P8:** [Pi JSON event stream](https://github.com/earendil-works/pi/blob/d981de1229ef899957bbe968bc8dcda02a21f477/packages/coding-agent/docs/json.md).
