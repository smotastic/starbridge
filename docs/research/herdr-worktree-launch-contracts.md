# Herdr 0.8.2 worktree and launch contracts

Research for [Verify Herdr worktree creation and launch confirmation contracts](https://github.com/smotastic/starbridge/issues/18). This note informs [Choose GitHub, Git, and Herdr integration methods](https://github.com/smotastic/starbridge/issues/15).

## Scope and evidence boundary

- Installed client: `herdr 0.8.2`.
- Installed server: `0.8.2`, protocol `20`, compatible.
- Installed pi integration: version `6`; Herdr 0.8.2 publishes integration version `8`. The installed integration is therefore outdated.
- Pinned Herdr source: commit [`9eb521456ac0d19d3ab3d9d7cea3cca10baa8a4c`](https://github.com/herdrdev/herdr/tree/9eb521456ac0d19d3ab3d9d7cea3cca10baa8a4c), release `v0.8.2`.
- Executed checks: read-only `--help`, `api schema --json`, `integration status`, and `status server`.
- No model prompt, worktree, session, integration install, server restart, or existing-session change was performed.

Command help and schema are current local evidence. Source links below establish behavior that help does not show. This research did not test a real create, launch, prompt, collision, timeout, or partial failure.

## Worktree creation contract

Use:

```text
herdr worktree create \
  --cwd <repository-root> \
  --branch <new-branch> \
  --base <fetched-commit> \
  --path <absolute-worktree-path> \
  --no-focus
```

`--workspace` and `--cwd` are alternatives. The CLI always prints one JSON response. The hidden `--json` option changes nothing.

The command first creates the parent directory. It then runs Git in a background operation. For a branch that does not exist locally, Herdr runs:

```text
git -C <repository-root> worktree add -b <branch> <path> <base>
```

For a branch that already exists locally, Herdr runs:

```text
git -C <repository-root> worktree add <path> <branch>
```

Therefore Starbridge must use a new branch or reject an existing branch before creation. An existing branch causes `--base` to be ignored. This is important for the exact fetched-commit requirement.

A successful response has this shape:

```json
{
  "id": "...",
  "result": {
    "type": "worktree_created",
    "workspace": { "workspace_id": "...", "active_tab_id": "..." },
    "tab": { "tab_id": "..." },
    "root_pane": { "pane_id": "...", "terminal_id": "..." },
    "worktree": {
      "path": "...",
      "branch": "...",
      "is_linked_worktree": true,
      "is_detached": false,
      "is_bare": false,
      "is_prunable": false,
      "label": "..."
    }
  }
}
```

The exact schema requires the fields above, with additional workspace, tab, pane, and worktree fields. The worktree response has no commit SHA. After a confirmed response, Starbridge must run read-only checks in the returned path:

```text
git -C <returned-path> rev-parse HEAD
git -C <returned-path> branch --show-current
git -C <repository-root> worktree list --porcelain
```

It must compare the returned path, branch, and `HEAD` with the intended values. Herdr's response is not enough to prove the commit.

## Worktree errors and unknown results

- Herdr rejects a second pending operation for the same canonical checkout path with `worktree_operation_in_progress`.
- Git errors are returned as `worktree_create_failed`. Herdr reports trimmed stderr, or stdout if stderr is empty.
- The create path is not preflighted for all collisions. Git decides whether an existing path or branch can be used.
- The create implementation does not remove a checkout after a failed Git command.
- If Git creates the checkout but Herdr cannot open or record the workspace, Herdr returns `worktree_open_failed` and leaves the Git worktree for inspection.
- The operation runs in a background thread. The API client waits for one response and has no default response timeout. If the client disconnects, the server may complete the operation but cannot return the result. Treat the result as unknown.

For an unknown result, do not retry creation. Inspect the repository with `git worktree list --porcelain`, the intended path, the intended branch, and `git -C <path> rev-parse HEAD`. Keep any created resource until the human inspects it.

Sources:

- [Worktree CLI](https://github.com/herdrdev/herdr/blob/9eb521456ac0d19d3ab3d9d7cea3cca10baa8a4c/src/cli/worktree.rs)
- [Worktree API and deferred operation](https://github.com/herdrdev/herdr/blob/9eb521456ac0d19d3ab3d9d7cea3cca10baa8a4c/src/app/api/worktrees/deferred.rs)
- [Git command construction and result handling](https://github.com/herdrdev/herdr/blob/9eb521456ac0d19d3ab3d9d7cea3cca10baa8a4c/src/worktree.rs)
- [Worktree schema](https://github.com/herdrdev/herdr/blob/9eb521456ac0d19d3ab3d9d7cea3cca10baa8a4c/src/api/schema/worktrees.rs)

## Agent start contract

Use the returned root pane:

```text
herdr agent start <name> \
  --kind pi \
  --pane <root-pane-id> \
  --timeout <remaining-startup-ms> \
  -- <pi-arguments>
```

The pane must contain an available interactive shell. Herdr starts the canonical `pi` executable in that pane. The startup timeout must be greater than 3000 ms and no more than 300000 ms. The default is 30000 ms.

The low-level API starts the process and returns `agent_started` with:

- `result.agent`: workspace, tab, pane, terminal, agent name, status, readiness flags, and state sequence;
- `result.argv`: the executable and arguments Herdr submitted.

The CLI adds readiness checks before it returns success. It confirms the same terminal, the requested name, detected kind `pi`, and `interactive_ready: true` while status is `idle` or `done`. It reports an error if the agent is blocked during startup, exits before interactive readiness, changes terminal, loses its name, has a kind mismatch, or reaches the timeout.

Record the returned server status, workspace ID, tab ID, pane ID, terminal ID, branch, worktree path, and `argv`. Herdr does not return the checkout commit in the start response. The separate Git checks are still required.

Sources:

- [Agent start CLI and readiness polling](https://github.com/herdrdev/herdr/blob/9eb521456ac0d19d3ab3d9d7cea3cca10baa8a4c/src/cli/agent.rs)
- [Agent start implementation](https://github.com/herdrdev/herdr/blob/9eb521456ac0d19d3ab3d9d7cea3cca10baa8a4c/src/app/agents.rs)
- [Agent schema](https://github.com/herdrdev/herdr/blob/9eb521456ac0d19d3ab3d9d7cea3cca10baa8a4c/src/api/schema/agents.rs)

## Prompt submission contract

Use the no-wait form:

```text
herdr agent prompt <agent-name> <prompt-text>
```

A successful response has `result.type` equal to `agent_prompted` and includes `result.agent`. Herdr rejects empty text, a blocked agent, a missing or unnamed agent, a pending managed startup, a missing terminal runtime, or a process that no longer matches the named agent.

Herdr writes the prompt to the terminal. It uses bracketed paste when the pane supports it, then sends Enter after a 300 ms delay. A successful response confirms that Herdr accepted and scheduled the terminal write. It does not confirm pi receipt, a pi turn, a conversation reference, or task execution.

Use no `--wait`. Waiting could finish an existing turn instead of the new prompt. A timeout, broken connection, missing response, or other uncertain result must not cause Starbridge to submit the prompt again. Inspect the retained pane, agent, pi session, and local launch record instead.

Sources:

- [Prompt CLI](https://github.com/herdrdev/herdr/blob/9eb521456ac0d19d3ab3d9d7cea3cca10baa8a4c/src/cli/agent.rs)
- [Prompt implementation](https://github.com/herdrdev/herdr/blob/9eb521456ac0d19d3ab3d9d7cea3cca10baa8a4c/src/app/api/agents.rs)
- [Prompt wait and stall behavior](https://github.com/herdrdev/herdr/blob/9eb521456ac0d19d3ab3d9d7cea3cca10baa8a4c/src/api/wait.rs)

## Session references and version boundary

Herdr's `AgentInfo` can include an optional `agent_session` object. Its required fields are `source`, `agent`, `kind`, and `value`. The bundled pi integration version 8 reports a pi session path or ID at `session_start`. The installed pi integration is version 6, so Starbridge must check the installed integration version during prerequisites and must not assume version-8 reporting.

The v8 integration sends state and session reports through short socket attempts. It treats any received socket data as delivery. It has no durable outbox. The installed v6 file has the same short retry pattern and also sends a release report on a `quit` event. These reports are useful references, but they are not launch acknowledgements and do not prove process lifetime.

Sources:

- [Herdr v0.8.2 bundled pi integration v8](https://github.com/herdrdev/herdr/blob/9eb521456ac0d19d3ab3d9d7cea3cca10baa8a4c/src/integration/assets/pi/herdr-agent-state.ts)
- Installed file: `/home/pi/.pi/agent/extensions/herdr-agent-state.ts`

## Remaining target checks

These facts are source-backed, but Starbridge still needs controlled checks in its own test replacements and one approved real demonstration:

1. Assert the exact JSON response fields and error codes for create, start, and prompt.
2. Check worktree creation against an existing path and branch. Do not use a real Starbridge path.
3. Check behavior when Herdr creates a worktree but workspace opening or response delivery fails.
4. Check the returned worktree `HEAD` against the fetched commit.
5. Check startup readiness in the intended dedicated worktree without sending a model prompt.
6. Check prompt success with a harmless test replacement, and record that this is not proof of pi task receipt.
7. Check that closing the Herdr client request does not stop the retained pane or worktree. Treat the result as unknown if the response is lost.
8. Check the installed integration version and either require the supported version or use a contract that does not depend on its telemetry.

No target check was run during this research because the ticket forbids worktree/session creation and prompt submission.
