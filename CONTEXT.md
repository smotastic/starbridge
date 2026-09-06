# Starbridge

Starbridge starts authorized repository work in a coding-agent session, then hands control to the human. It does not supervise the work after launch.

## Language

**Mission**:
One authorized backlog task and the work performed for it by a coding agent. Its progress after launch belongs to the agent and human, not Starbridge.

**Run**:
One invocation of Starbridge that attempts to launch at most one new mission, then exits. It does not wait for the mission's result.

**Eligible issue**:
An open, human-authorized backlog item that is not already marked as started, awaiting human attention, or handed off.

**Started mission**:
A mission marked for dispatch so later runs do not automatically launch it again. This marker does not prove that launch succeeded or that an agent is still running.

**Awaiting-input mission**:
An unfinished mission whose agent has requested human attention. Its session may remain open, and the human resumes it directly.
_Avoid_: Held mission (previously implied confirmed agent termination)

**Mission worktree**:
The dedicated repository workspace belonging to a mission, retained for agent work and manual inspection, resumption, or cleanup.

**Agent-reported handoff**:
A pull request and check summary supplied by the agent for human review. Starbridge has not independently verified the result; handoff does not mean approval or merge.
_Avoid_: Validated completion, supervised completion
