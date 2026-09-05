# Starbridge

Starbridge supervises authorized repository work performed by a coding agent, from issue selection to validated pull-request handoff.

## Language

**Mission**:
One authorized backlog task and its supervised progress toward a validated pull request. A mission can remain unfinished without currently executing.

**Executing mission**:
The mission currently permitted to run its coding agent; at most one exists per repository.
_Avoid_: Active mission (ambiguous between executing and unfinished)

**Held mission**:
An unfinished mission awaiting human attention, with its workspace retained. Its coding agent must be stopped before another mission executes.

**Completed mission**:
A mission whose validated pull request has been handed off for human review. Completion does not mean the change has been merged.

**Run**:
One invocation of the supervisor that may try multiple missions but ends after one successful pull-request handoff, no eligible work remains, or a supervisor-level failure.

**Eligible issue**:
A human-authorized backlog item ready for agent work and not marked as requiring human attention.

**Mission worktree**:
The isolated repository workspace belonging to a mission, which may be retained after execution stops for inspection or recovery.

**Validated pull request**:
A proposed change for which the configured local repository checks have passed. This does not imply successful remote CI, human approval, or merge.
