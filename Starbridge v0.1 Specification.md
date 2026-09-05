# Starbridge v0.1 Specification

## 1. Overview

Starbridge is a lightweight orchestration tool that allows an AI coding agent to autonomously work on a software repository for extended periods of time.

The human defines:

- architecture
- technical constraints
- project conventions
- policies
- specifications
- approval boundaries

Starbridge then autonomously selects work, launches exactly one coding agent, supervises its progress, and advances the repository through a controlled development workflow.

The primary deployment target is a Raspberry Pi or similar always-on machine.

Starbridge itself does not provide a full agent UI. Agent execution, observation, interactive control, and worktree-based isolation are delegated to Herdr.

The initial integration targets are:

- GitHub as backlog and pull-request system
- Herdr as agent runtime and interactive UI
- `pi` as the coding agent
- Git worktrees for isolated mission workspaces
- Git as source-control mechanism
- local filesystem or SQLite for Starbridge runtime state

The system must use a hexagonal architecture so that GitHub, Herdr, backlog providers, agent runtimes, and user interfaces can be replaced later.

---

# 2. Technology Stack

Starbridge must be implemented in **TypeScript**.

The complete project should live inside a **monorepo**.

Recommended runtime:

- Node.js
- TypeScript
- a modern TypeScript package manager/workspace solution such as pnpm workspaces
- a monorepo structure that keeps domain, application, ports, adapters, CLI, and tests clearly separated

Avoid framework-heavy architecture unless it solves an immediate requirement.

The implementation should favor:

- strict TypeScript settings
- explicit interfaces
- dependency inversion
- simple dependency injection through a composition root
- unit-testable application services
- minimal runtime dependencies

Suggested compiler expectations:

```text
strict = true
noImplicitAny = true
noUncheckedIndexedAccess = true
```

Exact TypeScript configuration may be adjusted where justified.

---

# 3. Monorepo Structure

The project should be organized as a monorepo.

A recommended initial structure is:

```text
starbridge/
├── apps/
│   └── cli/
│
├── packages/
│   ├── domain/
│   ├── application/
│   ├── ports/
│   ├── adapter-github/
│   ├── adapter-herdr/
│   ├── adapter-git/
│   ├── adapter-sqlite/
│   └── test-support/
│
├── package.json
├── pnpm-workspace.yaml
├── tsconfig.base.json
└── README.md
```

Responsibilities:

```text
apps/cli
→ executable Starbridge CLI and composition root

packages/domain
→ domain entities, value objects, policies, invariants

packages/application
→ workflows and use cases

packages/ports
→ interfaces required by the application layer

packages/adapter-github
→ GitHub Issues and Pull Request adapters

packages/adapter-herdr
→ Herdr agent runtime and human-interaction adapters

packages/adapter-git
→ repository and Git worktree operations

packages/adapter-sqlite
→ persistent runtime state

packages/test-support
→ fake/in-memory adapters and shared test utilities
```

The exact package split may be simplified if needed, but architectural boundaries must remain explicit.

Packages must obey the dependency direction:

```text
adapters / CLI
      ↓
application
      ↓
domain
```

Ports may be placed in their own package or colocated with the application layer, but external implementations must not leak into the core.

---

# 4. Core Product Idea

Starbridge behaves like a persistent autonomous development supervisor.

Typical scenario:

1. The repository owner defines architecture and constraints.
2. Starbridge is installed and initialized inside the repository.
3. A scheduler invokes `starbridge run`, for example every evening at 20:00.
4. Starbridge checks the backlog.
5. It selects one task.
6. Starbridge creates a dedicated Git worktree for that mission.
7. Starbridge starts a new Herdr session inside that worktree.
8. Herdr launches `pi` as the coding agent.
9. `pi` receives the mission specification and repository context.
10. `pi` creates a specification and implementation plan.
11. `pi` implements the task using TDD.
12. Tests and validation are executed.
13. A pull request is created.
14. Starbridge evaluates merge policy.
15. Safe work may be merged automatically.
16. Risky work waits for human approval.
17. The mission worktree is cleaned up when appropriate.
18. Starbridge continues with the next task.
19. Only one task and one coding agent are active at any time.

The user can observe or interact with the running `pi` session through Herdr at any moment.

---

# 5. Design Principles

## 5.1 Thin orchestrator

Starbridge should not attempt to become another coding-agent framework.

Starbridge decides:

- what should happen next
- whether a transition is allowed
- when human approval is needed
- which external adapter should perform an operation
- which worktree belongs to the current mission

The coding agent decides:

- how to understand the code
- how to implement the task
- how to write tests
- how to refactor code
- how to solve technical implementation details within the provided constraints

---

## 5.2 `pi` is the initial coding agent

The initial coding-agent implementation is `pi`.

Starbridge must not invoke a generic configurable agent command in an overly abstract way in v0.1 unless required by the Herdr integration.

The initial flow is conceptually:

```text
Starbridge
→ Herdr
→ launch session in mission worktree
→ start `pi`
→ send mission instructions
```

However, the application layer must depend only on `AgentRuntimePort`.

`pi` is an infrastructure choice, not a domain concept.

A future implementation should be able to replace `pi` without changing Starbridge's workflow logic.

---

## 5.3 One agent at a time

v0.1 must be strictly sequential.

There must never be more than one active Starbridge mission per repository.

There must never be more than one coding-agent session controlled by a Starbridge run.

There must therefore also never be more than one active mission worktree owned by the current Starbridge execution unless a retained worktree is intentionally kept for a blocked/recoverable mission.

Time-to-completion is less important than simplicity, reliability, and traceability.

---

## 5.4 GitHub is the work ledger

Every implementation task should ultimately correspond to a backlog item.

Initially this means GitHub Issues.

If the backlog is empty, Starbridge may ask the agent to identify useful improvements.

However, the agent must not directly implement an invented feature.

Instead:

1. inspect repository
2. propose improvement
3. create a backlog item
4. process the backlog item through the normal workflow

This keeps autonomous work auditable.

---

## 5.5 Spec-driven development

Implementation must be spec-driven.

The workflow should be compatible with tools or methodologies such as:

- Spec Kit
- Wayfinder
- similar specification-driven workflows

The exact spec engine should not be deeply coupled to Starbridge.

A normal mission should broadly follow:

```text
task
→ clarify
→ specification
→ implementation plan
→ tests
→ implementation
→ verification
→ review
→ pull request
```

---

## 5.6 TDD by default

The coding agent should follow test-driven development whenever reasonable.

Expected sequence:

```text
understand expected behavior
→ add/update failing test
→ implement behavior
→ make test pass
→ refactor
→ run complete relevant test suite
```

Exceptions can be allowed for documentation, configuration, or work where automated tests are not meaningful.

---

## 5.7 Worktree isolation

Every mission must execute inside its own Git worktree.

The main repository checkout must not be used as the coding agent's mutable workspace.

This provides:

- isolation from the user's primary checkout
- cleaner recovery
- predictable branch ownership
- easier inspection
- reduced accidental interference
- straightforward cleanup after mission completion

The mission worktree is considered part of mission runtime state.

---

# 6. Terminology

Starbridge may internally use space-themed terminology, but source code and logs should remain understandable.

Recommended conceptual terminology:

| Starbridge term | Meaning |
|---|---|
| Mission | One backlog task / GitHub Issue |
| Vessel | Coding-agent session |
| Orders | Prompt or additional instructions |
| Mission Log | Execution history |
| Dock | Repository |
| Launch | Start agent |
| Recall | Stop agent |
| Bridge | Interactive control / observation interface |
| Hold | Waiting for human approval |

These names are optional in implementation APIs.

Clarity is more important than maintaining the metaphor everywhere.

---

# 7. Architecture

Starbridge must use hexagonal architecture / ports and adapters.

High-level structure:

```text
External systems
      │
      ▼
Adapters
      │
      ▼
Ports
      │
      ▼
Application
      │
      ▼
Domain
```

Dependencies must point inward.

The domain and application layers must not depend on GitHub, Herdr, `pi`, shell commands, HTTP APIs, SQLite-specific implementations, or other infrastructure details.

---

# 8. Layers

## 8.1 Domain

The domain layer contains business concepts and invariant rules.

Possible domain entities/value objects:

```text
Mission
MissionId
MissionStatus
Run
RunId
AgentSession
AgentStatus
Worktree
WorktreeId
Policy
ApprovalRequirement
RiskLevel
ValidationResult
Change
```

Possible mission states:

```text
queued
selected
preparing_workspace
planning
implementing
verifying
reviewing
waiting_for_human
merging
completed
failed
cancelled
```

Domain rules include:

- only one mission may execute at once
- only one agent session may be active
- each active mission owns exactly one worktree
- an agent must execute inside its mission worktree
- failed validation prevents merge
- required HITL prevents automatic merge
- completed missions must have a terminal state
- Starbridge should be resumable after interruption
- autonomous tasks must become backlog items before implementation

The domain must have no knowledge of GitHub, Herdr, or `pi`.

---

## 8.2 Application

The application layer contains orchestration and use cases.

Suggested use cases:

```text
InitializeRepository
RunStarbridge
SelectNextMission
CreateAutonomousMission
PrepareMissionWorkspace
ExecuteMission
SuperviseAgent
VerifyMission
ReviewMission
RequestHumanApproval
MergeMission
CompleteMission
CleanupMissionWorkspace
ResumeRun
StopRun
```

`RunStarbridge` is the main orchestration use case.

It should coordinate the state machine but delegate all external operations through ports.

---

# 9. Ports

## 9.1 BacklogPort

Responsible for discovering and managing work.

Example interface:

```text
list_open_items()
get_item(id)
create_item(...)
mark_in_progress(id)
mark_completed(id)
add_comment(id, message)
```

Initial implementation:

```text
GitHubIssuesAdapter
```

Future implementations may include:

```text
LinearBacklogAdapter
JiraBacklogAdapter
LocalBacklogAdapter
```

---

## 9.2 AgentRuntimePort

Responsible for coding-agent lifecycle.

Example interface:

```text
start_agent(context)
get_agent_status(session_id)
send_instruction(session_id, instruction)
read_output(session_id)
stop_agent(session_id)
```

`start_agent(context)` must include the mission workspace path.

Conceptually:

```text
start_agent({
  workingDirectory,
  mission,
  initialPrompt
})
```

Initial implementation:

```text
HerdrAgentRuntimeAdapter
```

The Herdr adapter must launch `pi` inside the mission worktree.

Possible future implementations:

```text
LocalAgentRuntimeAdapter
TmuxAgentRuntimeAdapter
DockerAgentRuntimeAdapter
RemoteAgentRuntimeAdapter
```

---

## 9.3 WorkspacePort

Worktree management should be modeled explicitly rather than hidden inside generic shell commands.

Example interface:

```text
create_worktree(mission)
get_worktree(mission_id)
worktree_exists(mission_id)
remove_worktree(mission_id)
list_managed_worktrees()
```

Initial implementation:

```text
GitWorktreeAdapter
```

A mission worktree should contain:

```text
mission id
branch
absolute path
creation timestamp
lifecycle state
```

This port may internally use Git commands but application code must not execute raw `git worktree` commands.

---

## 9.4 SourceControlPort

Responsible for repository-level source-control operations.

Example interface:

```text
get_default_branch()
get_current_revision()
create_branch(name)
get_diff(workspace)
is_clean(workspace)
commit(...)
push(...)
```

Initial implementation may call local Git.

Worktree creation itself should live behind `WorkspacePort`.

---

## 9.5 ChangeReviewPort

Responsible for proposed changes and merging.

Example interface:

```text
create_change(...)
get_change(...)
get_checks(...)
merge_change(...)
close_change(...)
```

Initial implementation:

```text
GitHubPullRequestAdapter
```

This port must remain separate from the backlog port even though both initially use GitHub.

---

## 9.6 VerificationPort

Responsible for project validation.

Example interface:

```text
run_tests(workspace)
run_lint(workspace)
run_typecheck(workspace)
run_project_checks(workspace)
```

All verification must execute inside the mission worktree.

Implementation may use repository configuration and shell commands.

---

## 9.7 HumanInteractionPort

Responsible for requesting and receiving human input.

Example interface:

```text
request_approval(context)
request_instruction(context)
has_response(request_id)
read_response(request_id)
```

Possible implementation:

```text
HerdrHumanInteractionAdapter
```

A future UI should be replaceable without changing the application layer.

---

## 9.8 StateStorePort

Responsible for persistent Starbridge runtime state.

Example interface:

```text
load_run()
save_run(run)
acquire_lock()
release_lock()
```

Initial implementation:

```text
SQLiteStateStoreAdapter
```

or a simple filesystem implementation if sufficient.

---

# 10. Herdr Integration

Herdr is the initial execution and observation environment.

For each new mission:

1. Starbridge creates a dedicated Git worktree.
2. Starbridge asks Herdr to create/start a session whose working directory is that worktree.
3. The session launches `pi`.
4. Starbridge sends the structured mission prompt to `pi`.
5. The user can inspect and control the same session through Herdr.
6. Starbridge periodically queries the Herdr session state.
7. The session remains associated with the mission until completion, failure, cancellation, or explicit cleanup.

The required conceptual operation is:

```text
herdr + worktree + pi
```

The exact Herdr CLI/API invocation should be isolated inside `HerdrAgentRuntimeAdapter`.

Application code must not depend on Herdr command syntax.

---

# 11. Worktree Lifecycle

The worktree lifecycle is a first-class part of the workflow.

High-level flow:

```text
MISSION SELECTED
    ↓
CREATE BRANCH
    ↓
CREATE WORKTREE
    ↓
START HERDR SESSION IN WORKTREE
    ↓
START PI
    ↓
IMPLEMENT
    ↓
VERIFY
    ↓
PUSH / PR
    ↓
MERGE OR HOLD
    ↓
STOP SESSION
    ↓
REMOVE WORKTREE WHEN SAFE
```

Suggested worktree path convention:

```text
<starbridge-runtime-root>/worktrees/<mission-id>/
```

Example:

```text
~/.local/share/starbridge/worktrees/repo-name/issue-42/
```

The exact location should be configurable.

The worktree must not accidentally be created inside another worktree.

---

# 12. Worktree Cleanup Rules

A mission worktree may be removed when:

```text
mission completed
AND changes safely persisted remotely or intentionally discarded
AND no active agent session references the worktree
```

Do not automatically remove the worktree when:

- the mission failed unexpectedly
- human review is required
- uncommitted changes exist
- recovery may require inspecting the workspace
- the remote branch or PR has not been safely created

Blocked or failed worktrees should remain inspectable until explicitly resolved or cleaned up.

A later command may support:

```bash
starbridge cleanup
```

---

# 13. Adapters

Suggested structure:

```text
starbridge/
├── apps/
│   └── cli/
│
├── packages/
│   ├── domain/
│   ├── application/
│   ├── ports/
│   ├── adapter-github/
│   ├── adapter-herdr/
│   ├── adapter-git/
│   ├── adapter-sqlite/
│   └── test-support/
```

The composition root is the only place that should know which concrete adapters are selected.

Example:

```text
BacklogPort          = GitHubIssuesAdapter
AgentRuntimePort     = HerdrAgentRuntimeAdapter
WorkspacePort        = GitWorktreeAdapter
SourceControlPort    = GitAdapter
ChangeReviewPort     = GitHubPullRequestAdapter
StateStorePort       = SQLiteStateStoreAdapter
HumanInteractionPort = HerdrHumanInteractionAdapter
```

---

# 14. Repository Initialization

Starbridge should support:

```bash
starbridge init
```

The command initializes repository-local Starbridge configuration.

Suggested structure:

```text
.starbridge/
├── constitution.md
├── architecture.md
├── policies.yaml
└── config.yaml
```

Runtime state should not be committed.

---

# 15. Constitution

`.starbridge/constitution.md` defines non-negotiable project rules.

Examples:

- architecture conventions
- dependency rules
- test expectations
- preferred libraries
- forbidden technologies
- code style
- security rules
- API compatibility requirements
- expected documentation
- TDD requirement
- maximum acceptable scope for autonomous changes

`pi` must be instructed to treat the constitution as authoritative.

---

# 16. Architecture Document

`.starbridge/architecture.md` describes intended project architecture.

Possible content:

- major components
- dependency boundaries
- domain concepts
- external systems
- architectural decisions
- folder structure
- allowed dependency direction

`pi` must respect this architecture and should not introduce major architectural changes autonomously unless policy explicitly permits it.

---

# 17. Configuration

Example `.starbridge/config.yaml`:

```yaml
backlog:
  adapter: github

agent_runtime:
  adapter: herdr
  agent: pi

workspace:
  adapter: git-worktree

change_review:
  adapter: github

state:
  adapter: sqlite

human_interaction:
  adapter: herdr

workflow:
  spec_driven: true
  tdd: true
  max_parallel_missions: 1
```

Adapter-specific configuration should remain outside the domain layer.

Sensitive credentials must not be committed.

---

# 18. Policies

`.starbridge/policies.yaml` controls autonomous behavior.

Example:

```yaml
merge:
  auto_merge: true

risk:
  require_human_approval:
    - authentication
    - authorization
    - database_migration
    - deployment
    - infrastructure
    - billing
    - secrets
    - public_api_breaking_change

limits:
  max_files_changed: 30
  max_lines_changed: 1500

verification:
  require_tests: true
  require_lint: true
  require_typecheck: true
```

Exact values are configurable.

The policy engine should remain simple in v0.1.

---

# 19. Main Workflow

`starbridge run` starts the autonomous loop.

High-level workflow:

```text
START
  ↓
ACQUIRE LOCK
  ↓
LOAD CONFIGURATION
  ↓
LOAD/RECOVER STATE
  ↓
SYNC REPOSITORY
  ↓
SELECT MISSION
  ↓
CREATE BRANCH
  ↓
CREATE MISSION WORKTREE
  ↓
START HERDR SESSION IN WORKTREE
  ↓
START PI
  ↓
SPECIFY
  ↓
PLAN
  ↓
IMPLEMENT
  ↓
VERIFY IN WORKTREE
  ↓
CREATE PR
  ↓
EVALUATE POLICY
  ├─ approval required
  │      ↓
  │ WAITING_FOR_HUMAN
  │
  └─ approval not required
         ↓
       MERGE
         ↓
      COMPLETE
         ↓
    STOP SESSION
         ↓
    CLEAN WORKTREE
         ↓
    SELECT NEXT MISSION
```

---

# 20. Backlog Selection

v0.1 may use a simple deterministic priority.

For example:

1. explicitly prioritized issue
2. oldest eligible issue
3. oldest normal issue

Issues must be filtered for suitability.

Starbridge should be able to ignore issues marked with labels such as:

```text
blocked
wontfix
needs-design
manual-only
```

Exact labels should be configurable.

---

# 21. Empty Backlog Behavior

If no eligible issues exist, Starbridge enters exploration mode.

The exploration agent should also run through Herdr and `pi`.

Prefer running exploration inside a disposable worktree rather than the user's main checkout.

Workflow:

```text
BACKLOG EMPTY
    ↓
CREATE EXPLORATION WORKTREE
    ↓
START HERDR + PI
    ↓
INSPECT REPOSITORY
    ↓
IDENTIFY ONE USEFUL IMPROVEMENT
    ↓
WRITE PROPOSAL
    ↓
SELF-REVIEW PROPOSAL
    ↓
CREATE BACKLOG ITEM
    ↓
STOP EXPLORATION SESSION
    ↓
REMOVE EXPLORATION WORKTREE
    ↓
PROCESS NEW ITEM USING NORMAL WORKFLOW
```

The autonomous agent must not silently implement arbitrary features without creating a backlog item.

---

# 22. Agent Prompt

Each mission should receive structured context.

The prompt should include at least:

```text
repository context
mission / issue
acceptance criteria
constitution
architecture
relevant policies
required workflow
TDD requirement
specification requirement
current branch
worktree path
definition of done
instructions for what to do when blocked
```

The agent should explicitly be told:

```text
You are running as `pi` inside a dedicated Git worktree owned by Starbridge.
Do not leave this worktree.
Do not switch to unrelated branches.
Do not start unrelated work.
Do not modify Starbridge runtime state.
Do not bypass approval or merge policies.
```

---

# 23. Agent Supervision

Starbridge should periodically inspect the state of the Herdr-managed `pi` session.

Possible states:

```text
working
idle
blocked
completed
failed
```

When `working`:

```text
continue waiting
```

When unexpectedly `idle`:

Starbridge may inspect recent output and send a continuation instruction.

Example intent:

```text
Continue working on the current mission. Review the mission specification,
current worktree state and remaining tasks. Do not start unrelated work.
```

When `blocked`:

Starbridge should determine whether the blocker can be handled automatically.

If not:

```text
mission → WAITING_FOR_HUMAN
```

---

# 24. Pull Request Workflow

Every completed mission should produce a pull request unless configuration explicitly allows another workflow.

All branch and PR operations must refer to the mission worktree/branch.

The PR should contain:

- mission reference
- summary
- implementation details
- tests added/updated
- validation performed
- known risks
- relevant specification reference

Starbridge should then evaluate:

```text
tests
lint
typecheck
repository-defined checks
policy
risk
human-approval requirement
```

---

# 25. HITL

Human-in-the-loop must be supported as a first-class workflow state.

Example:

```text
REVIEWING
   ↓
policy.requires_human_approval == true
   ↓
WAITING_FOR_HUMAN
```

While waiting:

- no second mission starts
- the mission worktree remains available
- the Herdr session may remain available
- the Starbridge run remains resumable
- user can inspect `pi`
- user can inspect the PR
- user can send additional instructions
- user can approve or reject

Possible later CLI:

```bash
starbridge approve
starbridge reject
starbridge resume
```

For v0.1, human interaction may primarily happen through Herdr and GitHub.

---

# 26. Auto-Merge

Auto-merge should be allowed only when all required conditions are satisfied.

Minimum conditions:

```text
mission completed
AND required tests passed
AND required validation passed
AND PR is mergeable
AND no human approval required
AND policy allows auto-merge
```

Risky areas should require human approval by default.

Examples:

```text
authentication
authorization
secrets
billing
database migrations
deployment configuration
infrastructure
security-sensitive code
breaking public APIs
major architecture changes
```

---

# 27. CLI

The CLI is the main Starbridge interface.

Required v0.1 commands:

```bash
starbridge init
starbridge run
starbridge status
starbridge stop
```

Useful additional commands:

```bash
starbridge run --once
starbridge run --until 07:00

starbridge missions
starbridge mission show <id>

starbridge resume
starbridge inspect
starbridge cleanup
```

`starbridge run` means:

> Continue autonomous development until the process is stopped, the configured end condition is reached, or Starbridge requires human intervention.

`starbridge run --once` means:

> Execute exactly one mission and exit.

`starbridge status` should show at least:

```text
run status
current mission
mission state
agent state
Herdr session
worktree path
branch
pull request
whether human input is required
last meaningful event
```

---

# 28. Scheduling

Starbridge itself must not contain cron-specific scheduling logic.

Scheduling is external.

Example:

```cron
0 20 * * * cd /path/to/repository && starbridge run
```

The architecture should also allow later triggering through:

```text
systemd timer
manual CLI
GitHub webhook
external scheduler
other automation
```

The scheduler decides when Starbridge runs.

Starbridge decides what happens after invocation.

---

# 29. Single-Instance Lock

`starbridge run` must obtain a repository-specific lock before doing any work.

Pseudo-flow:

```text
starbridge run
  ↓
acquire repository lock
  ↓
lock already held?
  ├─ yes → exit without launching another agent
  └─ no  → continue
```

This prevents overlapping cron executions.

The lock must be released on clean shutdown.

The system should also handle stale locks after crashes.

---

# 30. Persistence and Recovery

Starbridge must be restartable.

The system should persist enough state to recover:

```text
run id
mission id
mission state
agent session id
worktree path
branch
PR id/URL
timestamps
approval state
last known agent status
```

After restart, Starbridge should inspect actual external state instead of assuming persisted state is still correct.

For example:

```text
saved state says IMPLEMENTING
→ inspect Herdr session
→ inspect mission worktree
→ inspect Git branch
→ inspect GitHub PR
→ reconcile state
→ continue safely
```

Recovery must never blindly create a second worktree or second `pi` session for the same active mission.

---

# 31. Logging

Every important transition should produce structured logs.

Example:

```text
run_started
mission_selected
worktree_created
herdr_session_started
pi_started
mission_planning
mission_implementing
verification_started
verification_failed
pull_request_created
human_approval_required
merge_started
mission_completed
herdr_session_stopped
worktree_removed
run_stopped
```

Logs should include:

```text
timestamp
run_id
mission_id
event
state
worktree
session_id
short message
```

Avoid relying exclusively on free-form `pi` output for system observability.

---

# 32. Graceful Stop

`starbridge stop` should request a controlled stop.

Preferred semantics:

```text
do not start another mission
finish current safe atomic action
persist state
leave Herdr session and worktree in recoverable state
exit
```

Do not destroy the worktree during graceful shutdown if the mission is incomplete.

A force-stop mechanism may be added separately later.

---

# 33. Out of Scope for v0.1

Do not implement the following unless required for the minimal system:

- multiple simultaneous agents
- multiple simultaneous missions
- custom web UI
- distributed execution
- Kubernetes
- complex workflow DSL
- generic plugin marketplace
- multi-repository orchestration
- agent-to-agent communication
- elaborate AI planning system inside Starbridge
- custom LLM provider abstraction beyond the Herdr/`pi` boundary
- sophisticated project-management replacement
- autonomous major architecture redesign
- long-term agent memory system

Prefer the smallest system that can successfully work autonomously overnight.

---

# 34. Key Architectural Constraint

The application layer must depend on capabilities, not products.

Bad:

```text
RunStarbridge → GitHubClient
RunStarbridge → HerdrClient
RunStarbridge → PiClient
```

Good:

```text
RunStarbridge → BacklogPort
RunStarbridge → AgentRuntimePort
RunStarbridge → WorkspacePort
RunStarbridge → ChangeReviewPort
```

Concrete products must only appear in adapters and composition/bootstrap code.

---

# 35. Initial Implementation Strategy

Implement in thin vertical slices.

## Slice 1 — Monorepo and initialization

Set up:

```text
TypeScript
pnpm workspace
strict TS config
test runner
linting
CLI package
core package boundaries
```

Support:

```bash
starbridge init
```

---

## Slice 2 — Read backlog

Support:

```bash
starbridge missions
```

Using `BacklogPort` and the GitHub adapter.

---

## Slice 3 — Worktree lifecycle

Implement `WorkspacePort`.

Support:

```text
create mission branch
create dedicated Git worktree
discover existing mission worktree
remove safe worktree
recover existing worktree
```

Test this independently of Herdr.

---

## Slice 4 — Launch one mission

Support:

```bash
starbridge run --once
```

Flow:

```text
select GitHub issue
→ create mission branch
→ create worktree
→ start Herdr session in worktree
→ launch pi
→ pass issue/context
```

Do not auto-merge yet.

---

## Slice 5 — Supervision

Track Herdr/`pi` lifecycle and persist mission state.

---

## Slice 6 — Verification and PR

Detect completion, execute repository checks inside the worktree, push branch, and open PR.

---

## Slice 7 — Policy and auto-merge

Add policy evaluation and HITL.

---

## Slice 8 — Continuous mode

Support:

```bash
starbridge run
```

Repeatedly execute missions sequentially.

---

## Slice 9 — Autonomous backlog generation

If the backlog is empty, create a temporary worktree, launch `pi` through Herdr, generate one candidate issue, clean up the exploration workspace, and process the new issue normally.

---

# 36. Testing Strategy

Architecture should make most behavior testable without GitHub, Herdr, Git worktrees, or `pi`.

Application tests should use fake/in-memory adapters.

Example:

```text
FakeBacklog
FakeAgentRuntime
FakeWorkspace
FakeSourceControl
FakeChangeReview
FakeStateStore
```

Core workflow tests should verify scenarios such as:

### Normal autonomous mission

```text
issue exists
→ selected
→ worktree created
→ agent starts in correct worktree
→ agent completes
→ tests pass
→ policy allows merge
→ PR merged
→ session stopped
→ worktree removed
→ mission completed
```

### Agent workspace isolation

```text
mission starts
→ dedicated worktree exists
→ Herdr receives worktree as cwd
→ pi executes inside worktree
→ primary repository checkout remains untouched
```

### Failed verification

```text
agent completes
→ tests fail
→ no merge
→ worktree retained
→ mission not marked completed
```

### HITL

```text
agent completes risky change
→ validation passes
→ policy requires approval
→ WAITING_FOR_HUMAN
→ Herdr session/worktree remain inspectable
→ no next mission starts
```

### Empty backlog

```text
no issues
→ exploration worktree
→ Herdr + pi
→ issue created
→ exploration worktree cleaned
→ issue selected
→ normal workflow
```

### Concurrent invocation

```text
run A holds lock
→ run B starts
→ run B exits
→ no second worktree
→ no second pi session
```

### Crash recovery

```text
mission implementing
→ process crashes
→ Starbridge restarted
→ existing worktree discovered
→ existing Herdr session discovered
→ same mission resumed
→ no duplicate worktree
→ no duplicate pi session
```

---

# 37. Definition of Done for v0.1

v0.1 is successful when the following scenario works reliably:

1. Starbridge is implemented in TypeScript.
2. The project is structured as a monorepo.
3. Repository contains Starbridge configuration.
4. Raspberry Pi runs a scheduled `starbridge run`.
5. Starbridge selects an eligible GitHub Issue.
6. Starbridge creates a dedicated Git worktree for that mission.
7. Exactly one Herdr session is launched in that worktree.
8. Herdr starts `pi`.
9. The user can inspect `pi` through Herdr.
10. `pi` follows project architecture and specification.
11. `pi` implements the issue inside its worktree.
12. Automated validation runs inside that worktree.
13. A pull request is created.
14. Safe changes may be automatically merged.
15. Unsafe or policy-sensitive changes wait for human review.
16. State survives a Starbridge process restart.
17. Existing Herdr sessions and worktrees can be reconciled after restart.
18. No second agent or mission worktree is launched while another mission is active.
19. Completed mission worktrees are safely cleaned up.
20. After completion, Starbridge may proceed to the next mission.
21. When the backlog is empty, Starbridge may use `pi` to generate one new backlog proposal before proceeding.

The key proof of the product is:

> Starbridge can be started in the evening, left unattended, and use Herdr plus `pi` inside isolated Git worktrees to produce a traceable, policy-compliant software change by the next morning without requiring the user to supervise the coding process.

---

# 38. Implementation Priority

Optimize v0.1 for:

1. reliability
2. recoverability
3. workspace isolation
4. understandable state transitions
5. traceability
6. strict architectural boundaries
7. safe autonomous execution

Do not optimize prematurely for:

```text
parallelism
throughput
complex scheduling
UI richness
framework flexibility
multi-agent coordination
```

The first version should remain deliberately small.

---

# 39. Instruction to the Coding Agent

When implementing Starbridge:

- use TypeScript
- structure the project as a monorepo
- use strict TypeScript settings
- follow hexagonal architecture strictly
- keep domain and application layers infrastructure-independent
- build through small vertical slices
- use TDD
- avoid speculative abstractions
- keep GitHub behind ports
- keep Herdr behind ports
- keep Git worktrees behind a dedicated workspace port
- treat `pi` as the initial concrete coding agent, not a domain concept
- launch `pi` through Herdr
- launch every mission in its own Git worktree
- never allow `pi` to modify the primary checkout
- implement one-agent-at-a-time semantics
- persist workflow state
- ensure interrupted runs are recoverable
- reconcile existing worktrees and Herdr sessions before creating replacements
- treat safety and HITL as normal workflow states rather than exceptions
- keep the CLI minimal
- do not build a custom web UI
- do not add multi-agent support in v0.1
- favor explicit state machines and deterministic behavior
- document architectural decisions when introducing important abstractions

Start by creating the TypeScript monorepo, core domain types, port interfaces, `WorkspacePort`, a minimal workflow state machine, and failing tests for the primary `starbridge run --once` workflow including worktree creation and Herdr launching `pi` inside that worktree.