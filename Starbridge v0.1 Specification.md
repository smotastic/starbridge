# Starbridge v0.1 Specification

## Status and source of truth

This is the agreed **starter** scope, not an implementation plan or a fully settled CLI contract. It replaces the original autonomous-supervisor specification and the earlier charting baseline where they conflict.

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

1. Acquires the repository launch lock, checks prerequisites, then selects and rechecks the oldest eligible open issue by creation time ascending, then issue number ascending for ties. Adding or restoring authorization does not change its position.
2. Creates the persistent local attempt record, then adds and confirms `starbridge:started` before creating the worktree or launching an agent.
3. Prepares a dedicated branch/worktree, never the primary checkout as the agent's mutable workspace.
4. Starts a new Herdr/pi session in that new dedicated Git worktree. The launch prompt supplies the issue reference, workspace, reporting obligations, and boundaries. It tells the agent to read the current issue and repository instructions, work on the issue, choose its workflow, run relevant checks, report questions or handoff, and never merge.
5. Records the Herdr session reference, branch, and worktree path in an issue comment.
6. Exits without waiting for a question, checks, or a PR.

A run attempts at most one new mission. It does not retry other issues after a failed launch or loop through blocked missions. If no eligible issue exists, it exits without inventing work.

After launch, sessions belong to the human rather than a Starbridge supervisor. A later CLI invocation can start another eligible issue even if earlier agents remain open or are still working. There is no repository-wide single-executing-agent guarantee, stop protocol, or human-intervention gate.

Support one configured launch host per repository. All runs for that repository, including different checkouts, share a host-managed launch lock. A competing run exits without launch changes. The lock covers launch and reporting and releases when Starbridge exits, including a crash; it does not cover the agent's lifetime. Separate launch hosts are unsupported. An external operation may continue after Starbridge exits; the retained started marker prevents later automatic selection of that issue.

## 5. Label contract

| Label | Meaning | Owner |
| --- | --- | --- |
| `starbridge:ready` | Human authorization for a new launch | Human |
| `starbridge:started` | Already marked for dispatch; exclude from future automatic selection | Starbridge before launch; manual recovery only |
| `starbridge:hitl` | Mission needs human attention, including a failed or uncertain launch | Agent or Starbridge adds; human removes during resumption or deliberate reset |
| `starbridge:handoff` | Agent reports work ready for human review, not independently verified success | Agent |

Eligibility requires an open issue with `starbridge:ready` and none of `starbridge:started`, `starbridge:hitl`, or `starbridge:handoff`. Exclusion markers always win, even for inconsistent combinations such as ready + handoff without started; Starbridge does not repair labels. Closed issues are excluded. Reopening clears no markers: a reopened ready issue without exclusions is eligible, while a reopened started issue remains excluded.

These conventions are settled in [Define starter eligibility and manual resumption conventions](https://github.com/smotastic/starbridge/issues/5#issuecomment-5561557791).

`starbridge:started` remains through questions, manual resumption, and PR handoff. It is a duplicate-selection guard, not evidence of successful launch, liveness, process termination, or completion. It is not an atomic lock for concurrent CLI invocations.

If dispatch fails or crashes after the marker is written, retain it for manual inspection rather than automatically relaunching. The human checks what exists before deliberately resetting a failed dispatch. To reset, the human first inspects retained sessions/worktrees and ensures an earlier agent will not continue the same work, then deliberately removes exclusion labels and retains/adds `starbridge:ready`. Starbridge does not verify reset safety. Preserve previous comments and resource references, and add a reset comment explaining why a fresh dispatch is safe and what happened to earlier resources. Launch-failure records and reporting follow the contract in section 8.

`starbridge:handoff` replaces the earlier `starbridge:done` name, including in eligibility rules; v0.1 requires no old-label support. A possible later use of done with autonomous merging is outside this version, not a settled future design. Handoff removes ready and always retains started, in the order specified below.

## 6. Questions and manual resumption

Starbridge also attempts to add `starbridge:hitl` after a failed or uncertain launch. In that case an agent session may not exist. The human inspects the issue, any retained Herdr/pi session, and local launch logs before resuming or resetting dispatch.

When blocked or requesting permission, the agent first posts an issue comment with the blocker, relevant evidence, and the specific answer or action needed, then adds `starbridge:hitl`. It must not guess permission or continue work dependent on the missing answer. Its Herdr/pi session and worktree may remain available; no confirmed stopping is required.

The human supplies the answer, resumes the existing agent directly in Herdr, and manually removes `starbridge:hitl`. Removing the label does not trigger another launch because `starbridge:started` remains present.

The agent reads the current issue when beginning. Subsequent issue edits require the human to notify the existing session; edits neither trigger relaunch nor guarantee that the agent notices. Manual resumption continues that mission with started retained; a dispatch reset instead reauthorizes a fresh dispatch under the inspection guidance above.

Starbridge does not poll for replies, deliver answers, resume sessions, detect human interference, or require a gate before interaction. An awaiting-input mission does not prevent a subsequent run from starting another eligible issue.

## 7. PR handoff and trust boundary

The agent owns checks, commits, pushes, PR creation, and reporting. Repository instructions determine the workflow and checks; use existing repository scripts and check configuration as applicable. Report check commands and results, or explain why no applicable checks were found or run, including documentation-only work. Never report unrun checks as passed. The human reviews the PR and CI and merges manually.

Handoff requires completed requested work in a non-draft PR ready for human review. Failing checks are permitted if prominently disclosed; draft PRs, incomplete work, and missing PRs do not qualify. The agent asks for human input when unable to proceed. Starbridge does not independently execute checks, inspect the final revision, validate label claims, watch CI, or perform a repair loop.

The PR links to the issue and includes a change summary, check commands/results, and known failures or limitations. Publish in this order: push changes, create the review-ready PR, post an issue comment with the PR link and short check summary, add `starbridge:handoff`, then remove `starbridge:ready`. Always retain `starbridge:started`. The human still removes `starbridge:hitl` when resuming.

If commenting, pushing, PR creation, or label updates fail, preserve local work. Report the failed operation, what succeeded, what remains, and recovery guidance in the existing session; use an issue comment where possible. Never claim publication or handoff succeeded when it failed. Do not undo published work. No mandatory retry loop or automatic replacement session is required.

These instructions are settled in [Define the minimal agent question and PR handoff instructions](https://github.com/smotastic/starbridge/issues/6#issuecomment-5568359394).

A comment, label, idle terminal, or PR is not proof that all mission processes have stopped or that the work is correct. The system makes no such guarantee. If an agent crashes, stalls, or forgets to report, the issue can remain `starbridge:started` until the human inspects it.

The agent must not merge. This is an instruction in a trusted environment, not a claim of capability enforcement.

## 8. Launch and runtime feasibility

[Verify Herdr and pi capabilities required by the MVP](https://github.com/smotastic/starbridge/issues/2) records the versioned [runtime capability research](docs/research/herdr-pi-capabilities.md). Its launch, prompt delivery, workspace identity, retained-session, and ARM findings remain relevant. Its extension, stop-confirmation, and supervision-recovery options are historical alternatives, not MVP requirements.

The launch contract is settled in [Define the starter launch boundary and partial-failure behavior](https://github.com/smotastic/starbridge/issues/9#issuecomment-5600986986).

Launch success requires Herdr confirmation of pi readiness in the dedicated worktree, successful prompt submission, and GitHub confirmation of the comment with session, branch, and worktree references. This confirms launch steps, not exactly-once task receipt, continued agent activity, or task completion.

Record each external operation locally before starting it, then record its confirmed result. Retain the issue identity, times, intended resource names, returned references, and failure details. A missing result means the outcome is unknown and needs manual inspection. If a local record update fails, stop before the next launch step.

A failure stops further launch steps. After issue selection, attempt both an issue comment with confirmed steps, failures, uncertainty, references, and recovery guidance, and the `starbridge:hitl` label. Record failed GitHub reporting locally where possible. Before selection, only local reporting is possible.

If writing started fails or its response is uncertain, do not create a worktree or launch pi. Never remove a marker that may have been written. Later runs use normal eligibility rules. After started is confirmed, retain it and all created resources after any failure. Never automatically repeat uncertain prompt submission, launch a replacement, stop an agent, or clear the marker.

If prompt submission succeeds but publishing references fails, report `Prompt submitted; launch record incomplete` and return an unsuccessful command result. The agent may continue. A timeout does not prove that an operation had no effect.

The preferred inspection order is the GitHub issue, any retained Herdr/pi session, then persistent local launch logs. A GitHub outage or sudden crash can prevent issue updates; a crash can also prevent a final log entry. An operation recorded before the crash without a confirmed result remains unknown. These limits are accepted. No automatic GitHub repair runs later. Logs describe launch attempts, not current agent progress.

No agent-lifetime tracking, automatic reconciliation after a restart, or automatic replacement/resumption is required. Retained resources and incomplete dispatches are inspected and recovered manually.

## 9. CLI and scheduling

The core operation is `starbridge run`: attempt to launch one eligible issue, record the references, then exit.

A read-only `starbridge logs` command is required for the human or their agent to inspect persistent local launch records. It must not repair, resume, replay, or launch work. Its options, output, and log location remain for the CLI decision.

The remaining operator decision will determine the minimal setup/configuration, preflight checks, output, exit statuses, and launch-time bounds. Extra `init`, `status`, `stop`, `resume`, or cleanup commands are not inherited requirements from the original specification.

Scheduling stays external, for example manual invocation, cron, or a systemd timer. Starbridge does not manage mission-duration budgets or stop agents when a timer expires.

## 10. Remaining decisions and acceptance proof

The map's open child tickets are authoritative for remaining work. Areas still requiring decisions are:

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
