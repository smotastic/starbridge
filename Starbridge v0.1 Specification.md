# Starbridge v0.1 Specification

## Status and source of truth

This is the agreed **starter** scope, not an implementation plan or a fully settled launch contract. It replaces the original autonomous-supervisor specification and the earlier charting baseline where they conflict.

The canonical decision index is [Starbridge v0.1 — MVP scope and feasibility map](https://github.com/smotastic/starbridge/issues/1). The scope change was confirmed through [Choose a trustworthy mission supervision contract](https://github.com/smotastic/starbridge/issues/3#issuecomment-5560000957). Detailed decisions live in their tracker tickets; this document summarizes the product requirements. Remaining questions below must be settled before implementation planning.

## 1. Product

Starbridge is a small issue-to-agent starter for a personal, trusted repository on a Raspberry Pi.

A human authorizes an issue. Starbridge selects it, marks it as started, launches a Herdr/pi session in a dedicated Git worktree, passes the issue reference and generic reporting instructions, records the workspace/session references, and exits.

The agent then does the work independently. It either asks for input on the issue or opens a PR with a check summary. The human owns observation, intervention, resumption, recovery, review, merging, and cleanup.

Starbridge does not know whether an issue requests a specification, a feature, documentation, or another kind of repository work. The issue and repository instructions determine the workflow, not Starbridge.

## 2. Fixed constraints

- TypeScript with strict settings, in a monorepo.
- Hexagonal boundaries: domain and application logic do not depend on GitHub, Herdr, pi, or shell command syntax.
- Initial integrations: GitHub issues and PRs, Herdr launching pi, and dedicated Git worktrees.
- Raspberry Pi is the primary deployment target; existing capability evidence covers Linux aarch64, not every Pi/OS combination.
- Keep ports and package boundaries minimal. No mandatory supervisor state machine, SQLite store, verification adapter, or plugin framework.
- Worktrees provide workspace isolation, not a security sandbox. Only trusted repositories and explicitly authorized issues are supported.
- No pi-specific extension is required. GitHub comments, labels, and PRs are the reporting surface; Herdr and the ordinary pi TUI are the human interface.

## 3. Ownership

| Starbridge | Agent | Human |
| --- | --- | --- |
| Select one eligible issue | Read the issue and repository instructions | Authorize issues |
| Mark dispatch before launch | Choose and execute the task's workflow | Observe and interact through Herdr |
| Prepare a dedicated worktree and launch Herdr/pi there | Run repository checks and report results | Answer questions and resume the existing session |
| Pass the issue reference and generic reporting instructions | Comment and label when input is needed | Inspect ambiguous or failed launches |
| Record issue/session/branch/worktree references and exit | Commit, push, open a PR, and report handoff | Review CI and PRs, merge, and clean up |

Starbridge does not prescribe or track specification, planning, TDD, or implementation stages. Repository-specific requirements may still prescribe any of these to the agent.

## 4. One run

A normal invocation:

1. Selects the oldest eligible open issue by creation time ascending, then issue number ascending for ties. Adding or restoring authorization does not change its position.
2. Adds `starbridge:started` before launching an agent.
3. Prepares a dedicated branch/worktree, never the primary checkout as the agent's mutable workspace.
4. Starts a new Herdr/pi session in that worktree and passes the GitHub issue reference plus generic reporting instructions.
5. Records the Herdr session reference, branch, and worktree path in an issue comment.
6. Exits without waiting for a question, checks, or a PR.

A run attempts at most one new mission. It does not retry other issues after a failed launch or loop through blocked missions. If no eligible issue exists, it exits without inventing work.

After launch, sessions belong to the human rather than a Starbridge supervisor. A later CLI invocation can start another eligible issue even if earlier agents remain open or are still working. There is no repository-wide single-executing-agent guarantee, stop protocol, or human-intervention gate.

This permits overlapping agent sessions; it does not settle the separate question of simultaneous CLI invocations racing to dispatch the same issue. The launch-boundary decision must address that explicitly.

## 5. Label contract

| Label | Meaning | Owner |
| --- | --- | --- |
| `starbridge:ready` | Human authorization for a new launch | Human |
| `starbridge:started` | Already marked for dispatch; exclude from future automatic selection | Starbridge before launch; manual recovery only |
| `starbridge:hitl` | Agent needs human input | Agent adds; human removes when resuming |
| `starbridge:done` | Agent reports PR handoff, not independently verified success | Agent |

Eligibility requires an open issue with `starbridge:ready` and none of `starbridge:started`, `starbridge:hitl`, or `starbridge:done`. Exclusion markers always win, even for inconsistent combinations such as ready + done without started; Starbridge does not repair labels. Closed issues are excluded. Reopening clears no markers: a reopened ready issue without exclusions is eligible, while a reopened started issue remains excluded.

These conventions are settled in [Define starter eligibility and manual resumption conventions](https://github.com/smotastic/starbridge/issues/5#issuecomment-5561557791).

`starbridge:started` remains through questions, manual resumption, and PR handoff. It is a duplicate-selection guard, not evidence of successful launch, liveness, process termination, or completion. It is not an atomic lock for concurrent CLI invocations.

If dispatch fails or crashes after the marker is written, retain it for manual inspection rather than automatically relaunching. The human checks what exists before deliberately resetting a failed dispatch. To reset, the human first inspects retained sessions/worktrees and ensures an earlier agent will not continue the same work, then deliberately removes exclusion labels and retains/adds `starbridge:ready`. Starbridge does not verify reset safety. Preserve previous comments and resource references, and add a reset comment explaining why a fresh dispatch is safe and what happened to earlier resources. Exact launch-failure breadcrumbs and failure output remain to be decided.

The earlier convention that agent-reported PR handoff removes `starbridge:ready` remains the baseline. `starbridge:started` stays present regardless; exact agent update ordering remains for the reporting decision.

## 6. Questions and manual resumption

When blocked, the agent comments on the issue with the question or blocker and adds `starbridge:hitl`. Its Herdr/pi session and worktree may remain available; no confirmed stopping is required.

The human supplies the answer, resumes the existing agent directly in Herdr, and manually removes `starbridge:hitl`. Removing the label does not trigger another launch because `starbridge:started` remains present.

The agent reads the current issue when beginning. Subsequent issue edits require the human to notify the existing session; edits neither trigger relaunch nor guarantee that the agent notices. Manual resumption continues that mission with started retained; a dispatch reset instead reauthorizes a fresh dispatch under the inspection guidance above.

Starbridge does not poll for replies, deliver answers, resume sessions, detect human interference, or require a gate before interaction. An awaiting-input mission does not prevent a subsequent run from starting another eligible issue.

## 7. PR handoff and trust boundary

The agent owns checks, commits, pushes, PR creation, and reporting. It is instructed to run the relevant repository checks and include their results in its handoff. The human reviews the PR and CI and merges manually.

An agent-created PR and reported check summary are sufficient for the reduced MVP's handoff model. Starbridge does not independently execute checks, inspect the final revision, validate label claims, watch CI, or perform a repair loop.

A comment, label, idle terminal, or PR is not proof that all mission processes have stopped or that the work is correct. The system makes no such guarantee. If an agent crashes, stalls, or forgets to report, the issue can remain `starbridge:started` until the human inspects it.

The agent must not merge. This is an instruction in a trusted environment, not a claim of capability enforcement. How missing checks, failing checks, documentation-only tasks, or failed PR creation should be reported is still an open decision.

## 8. Launch and runtime feasibility

[Verify Herdr and pi capabilities required by the MVP](https://github.com/smotastic/starbridge/issues/2) records the versioned [runtime capability research](docs/research/herdr-pi-capabilities.md). Its launch, prompt delivery, workspace identity, retained-session, and ARM findings remain relevant. Its extension, stop-confirmation, and supervision-recovery options are historical alternatives, not MVP requirements.

Launch needs bounded, observable success/failure behavior. Herdr readiness or prompt submission is not proof that the task was received exactly once or eventually completed. The remaining launch decision must define the smallest honest CLI success criterion, partial-failure handling, and duplicate-launch precautions without reintroducing a supervisor.

No agent-lifetime tracking, automatic reconciliation after a restart, or automatic replacement/resumption is required. Retained resources and incomplete dispatches are inspected and recovered manually.

## 9. CLI and scheduling

The core operation is `starbridge run`: attempt to launch one eligible issue, record the references, then exit.

The remaining operator decision will determine the minimal setup/configuration, preflight checks, output, exit statuses, and launch-time bounds. Extra `init`, `status`, `stop`, `resume`, or cleanup commands are not inherited requirements from the original specification.

Scheduling stays external, for example manual invocation, cron, or a systemd timer. Starbridge does not manage mission-duration budgets or stop agents when a timer expires.

## 10. Remaining decisions and acceptance proof

The map's open child tickets are authoritative for remaining work. Areas still requiring decisions are:

- [Define the starter launch boundary and partial-failure behavior](https://github.com/smotastic/starbridge/issues/9).
- [Define the minimal agent question and PR handoff instructions](https://github.com/smotastic/starbridge/issues/6).
- [Set the minimal starter CLI and setup contract](https://github.com/smotastic/starbridge/issues/7).
- [Agree the starter release-proof scenarios and scope completeness](https://github.com/smotastic/starbridge/issues/10).

The product proof is a Raspberry Pi invocation that launches an authorized issue in its own Herdr/pi worktree, records usable references, and exits while the agent can continue. The human later finds a question or an agent-reported PR handoff. A subsequent invocation can select another eligible issue without relaunching an already-started one.

That is a target for acceptance testing, not an end-to-end certification already established by the research. Normal launch, input-needed, manual resumption, handoff, no eligible issue, and ambiguous launch paths need concrete acceptance coverage once their decisions settle. Do not start implementation from the superseded supervisor task breakdown.

## 11. Out of scope

- Long-running supervision, terminal scraping, heartbeat protocols, or a mandatory pi extension.
- Starbridge-controlled agent stopping, process-tree confirmation, single-agent execution enforcement, or human-intervention gates.
- Automatic mission restart, reconciliation, resumption, or retry/repair loops.
- Independent local verification, remote CI supervision, automatic merging, or PR review/revision automation.
- Workflow engines, mandatory spec/plan/TDD stages, and autonomous backlog generation.
- Continuous launch loops or multiple new missions per invocation.
- Automatic cleanup or a cleanup command; resources remain for manual inspection and cleanup.
- Custom web UI, distributed or multi-repository orchestration, agent-to-agent coordination, plugin marketplaces, and speculative framework abstractions.
- Hostile-repository sandboxing and enforcement of agent permissions.
- Detailed implementation planning and implementation during this Wayfinder effort.

Existing human-controlled sessions may run concurrently. Concurrency itself is not prohibited; Starbridge-managed coordination of those sessions is out of scope.
