# Design Proposal: Coding-Agent Mode (Embedded CLI + Bring-Your-Own-Workspace)

**Status:** proposal · **Depends on:** nothing upstream (additive) · **Relates to:**
[CONTRIBUTION_STRATEGY.md](CONTRIBUTION_STRATEGY.md) items #2 and #3

## Motivation

Phlox has a complete agentic harness — tool loop, approvals, checkpoints, sub-agents,
skills, sandboxed execution — but it can only be driven from the web UI, and every
conversation is jailed to a managed scratch directory (`data/workspaces/<conversation_id>/`).
That rules out the fastest-growing way people use agents: **open a terminal in the project
you're working on (e.g. the VS Code integrated terminal) and run the agent against that
code, in place** — the Claude Code / Codex CLI / kiro-cli experience.

Two coupled features close the gap:

1. **Bring-your-own-workspace (BYOW):** let a conversation's workspace be an existing
   directory instead of a managed scratch dir.
2. **Embedded CLI (`phlox agent`):** a terminal REPL that runs the existing harness
   in-process, with the workspace rooted at the current directory.

They are designed together but land as separate PRs — BYOW is useful on its own (point a
web conversation at an allowlisted project dir), and the CLI's Phase A (packaging/ops) is
useful before BYOW exists.

## Goals / non-goals

**Goals**

- `cd ~/code/myproject && phlox agent` gives a streaming, approval-aware agent session
  operating on the project in place, with no server running.
- Zero behavior change for existing deployments: BYOW is off by default and file-config
  gated; the CLI is a new entry point.
- Reuse the harness unchanged — same `AgentSession`, same tools, same `PermissionGate`,
  same event stream. The CLI is a *renderer* of the stream the web UI already consumes.
- All three sandbox runners keep working against a BYOW workspace (local, container,
  agentcore).

**Non-goals (v1)**

- A remote-client CLI (talking to a running Phlox server). That is Gateway Phase 2's
  client and a follow-up (Phase C below).
- IDE plugins/extensions. The integrated terminal *is* the VS Code integration, as with
  Claude Code.
- Multi-root workspaces, watch mode, or token-level steering.

## User experience sketch

```
$ cd ~/code/myproject
$ phlox agent
Phlox agent · model: qwen3-coder (ollama) · workspace: ~/code/myproject · sandbox: local
> add retry logic to the fetch helper in src/net.ts

⚙ read_file src/net.ts
⚙ edit_file src/net.ts        [ask] apply this edit? (y/n/a=always) y
⚙ run_shell npm test          [ask] run this command? (y/n/a) y
  │ > vitest run …
  │ ✓ 42 passed
Done — added exponential backoff (3 attempts) to fetchJson; tests pass.

> /diff        # git diff of the project since the session started
> /undo        # restore the previous shadow checkpoint
> /quit
```

Flags: `--model/-m`, `--profile`, `--auto-approve` (a.k.a. agent mode), `--continue`
(resume the most recent conversation for this directory), `--max-rounds`.

---

## Part 1 — Bring-your-own-workspace (backend)

### Current state

- `workspace/manager.py:13` — `workspace_dir(conversation_id)` returns (and creates)
  `WORKSPACES_DIR / conversation_id`; `resolve_in_workspace` jails all file paths under it.
- `agent/harness.py:74` — `AgentSession` hardcodes `self.workspace = workspace_dir(conversation.id)`.
- `routers/files.py` and `routers/checkpoints.py` resolve the workspace the same way.
- `workspace/checkpoints.py` makes each workspace a git repo and auto-commits
  (`git add -A && git commit`) after every mutating tool.

### Schema

Add one nullable column:

```python
class Conversation(Base):
    ...
    workspace_root: Mapped[str | None] = mapped_column(String(1024), nullable=True)
```

`NULL` (the default, and the only value creatable today) means "managed scratch dir" —
existing behavior is untouched. A non-null value is an absolute path to an existing
directory, validated at set time (see Security).

### Resolution seam

Replace direct `workspace_dir(conversation_id)` calls with a conversation-aware resolver
in `workspace/manager.py`:

```python
def resolve_workspace(conversation: Conversation) -> Path:
    if conversation.workspace_root:
        root = Path(conversation.workspace_root).resolve()
        _require_allowed(root)          # re-check the allowlist on every resolution,
        return root                     # not just at set time (config may have changed)
    return workspace_dir(conversation.id)
```

`resolve_in_workspace` gains a `root: Path` parameter (or a conversation-aware variant);
its traversal guard is unchanged — the jail simply re-roots at the project directory. Call
sites to update: `agent/harness.py:74`, `routers/files.py:28`,
`routers/checkpoints.py:19,24`, and the fs tools' path resolution.

### Setting it

- **API:** `workspace_root` accepted on conversation create (`POST /api/conversations`),
  immutable afterwards (changing the jail mid-conversation invalidates message history
  that references files).
- **Web UI (optional, later):** a "folder" picker on the new-chat screen, only shown when
  the server has a non-empty allowlist.
- **CLI:** sets it to `Path.cwd()` on conversation create (Part 2).

### Security model

BYOW deliberately relaxes two guarantees, so it is **off by default** and **file-config
only** (same rationale as `sandbox.runner`: flipping isolation at runtime from the UI
would be a security surprise — see [SANDBOX.md](SANDBOX.md)):

```yaml
workspace:
  allowed_roots: []        # empty (default) = BYOW disabled entirely
  # allowed_roots:
  #   - /home/alice/code   # any subdirectory of these may be mounted
```

Rules:

1. A requested `workspace_root` must resolve (symlinks followed) to a directory **under
   one of `allowed_roots`**; otherwise 403. Re-checked on every `resolve_workspace`, so
   removing a root from config immediately locks out conversations pointing into it.
2. Managed-workspace internals must never be mountable: reject roots under `DATA_DIR`.
3. **Multi-user note:** per-user data isolation does not apply inside a mounted directory
   — any user who may create a BYOW conversation on a root can read/write it. v1 keeps
   the allowlist global and therefore effectively single-tenant/trusted-team; a per-user
   allowlist is a follow-up if demand appears. This is stated in the docs, not silently
   implied.
4. The path-traversal guard (`resolve_in_workspace`) continues to apply verbatim; tools
   cannot escape the mounted root.

### Checkpoints: the landmine

`workspace/checkpoints.py` runs `git init` (if needed) and `git add -A && git commit` in
the workspace after every mutating tool. Against a real project this would **write Phlox
commits onto the user's current branch** — silently rewriting their history. Unacceptable.

v1 approach: **shadow repo, not the project's repo.** For BYOW workspaces, checkpoint
operations run with `GIT_DIR` pointed at a Phlox-owned bare repo
(`DATA_DIR/checkpoints/<conversation_id>.git`) and `GIT_WORK_TREE` at the project root, so
snapshots and one-click restore keep working without ever touching `.git/` in the project.
The existing per-workspace `RLock` and best-effort semantics carry over. The project's
`.gitignore` is honored (shadow commits use `git add -A` which respects it), which also
keeps `node_modules/` out of snapshots for free.

If the shadow-repo variant proves contentious upstream, the fallback is simpler: disable
auto-checkpointing when `workspace_root` is set and document "use your project's own git."
The proposal prefers the shadow repo because `/undo` is exactly what a coding agent needs.

### Sandbox runner interplay

| Runner | Behavior with BYOW | Change needed |
|---|---|---|
| `local` | Commands run rooted at the project dir — the Claude Code model | none |
| `container` | Project dir bind-mounted at `/work` | none |
| `agentcore` | Whole workspace synced into the microVM per run | **sync ignore rules** (below) |

**AgentCore sync ignore rules.** The runner syncs the full workspace in before each run
and changed files back out after — fine for scratch dirs, pathological for a repo with
`.git/`, `node_modules/`, or build output (and `max_sync_file_bytes`, default 5 MB,
silently skips big files). Add to the runner (useful independently of BYOW, so it can be
its own small PR):

```yaml
sandbox:
  agentcore:
    sync_ignore:               # gitwildmatch patterns, merged with built-in defaults
      - "*.parquet"
    respect_gitignore: true    # also apply the workspace's .gitignore
```

Built-in defaults: `.git/`, `node_modules/`, `.venv/`, `venv/`, `__pycache__/`, `dist/`,
`build/`, `.next/`, `target/`. Applied to sync-in, sync-back, and the baseline snapshot.
The same ignore set should bound artifact capture (`snapshot_dir`) for BYOW workspaces,
which otherwise walks the whole repo per tool call. For genuinely large repos the
documented recipe remains: clone inside the AgentCore session and push from there, so only
the diff crosses the wire.

---

## Part 2 — Embedded CLI (`phlox agent`)

### Shape

A new `backend/app/cli/` package plus entry point:

```toml
[project.scripts]
phlox = "app.cli.main:main"
```

Subcommand layout (Phase A ships `serve`/`init`/`admin`/`rag`; this proposal details
`agent`):

```
phlox serve                  # uvicorn wrapper (Phase A)
phlox init                   # scaffold config.yml + data dir (Phase A)
phlox admin create-user ...  # headless admin (Phase A)
phlox rag reindex            # (Phase A)
phlox agent [prompt]         # embedded coding-agent REPL (Phase B, this doc)
```

### Why embedded (in-process) rather than client/server

The backend is plain Python; `AgentSession` needs only a db session, a `Conversation`
row, a provider, the registry, and a `PermissionGate` — exactly what `spawn_subagent`
already constructs programmatically (`agent/tools/subagent.py`). Running in-process means:

- no server to start, no auth handshake, no port — `pipx install phlox` and go;
- tools execute on the user's machine in the user's directory, which is the *point* of a
  coding agent (a remote server's tools can't see local files);
- the provider config (Ollama, Bedrock, OpenAI-compatible) is read the same way the
  server reads it.

The remote-client mode (connect to a shared Phlox for server-side RAG/memory/budgets) is
Phase C, blocked on Gateway Phase 2, and slots in as an alternate transport behind the
same renderer.

### State: a real (local) conversation, not a throwaway

Embedded mode uses a per-user data dir — `PHLOX_DATA` already exists as an env override
(`config.py:23`), so the CLI defaults it to `~/.phlox/` and `PHLOX_CONFIG` to
`~/.phlox/config.yml` (scaffolded by `phlox init`). It then uses the normal `SessionLocal`
and creates a **persistent `Conversation` with `workspace_root=Path.cwd()`**. Everything
downstream — message history, `PendingApproval`, tool prefs, memories, skills, usage —
works unchanged because it *is* the normal stack. `phlox agent --continue` reopens the
most recent conversation whose `workspace_root` matches cwd, giving cheap session
resumption per project.

The BYOW allowlist applies here too, with one relaxation: in embedded mode the process
runs *as the user on their own machine* (the trust model of `sandbox.runner: local`), so
the CLI implicitly allows cwd. Config can still pin `workspace.allowed_roots` in
`~/.phlox/config.yml` to restrict it.

### Rendering the event stream

`AgentSession.run()` / `.resume()` yield SSE-framed JSON events (`agent/events.py`) — the
CLI consumes the same generator the chat router streams to the browser and renders per
event type:

| Event | Rendering |
|---|---|
| `token` | stream to stdout |
| `thinking` | dim/gray, collapsible via `--no-thinking` |
| `status`, `tool_call` | one-line tool header (`⚙ run_shell npm test`) |
| `tool_progress` | indented live output (the harness already streams stdout/stderr) |
| `tool_result` | compact result summary; `--verbose` for full content |
| `artifact` | print workspace-relative path |
| `approval_request` + `paused` | **interactive prompt** (below) |
| `usage`, `done` | one-line turn summary (tokens, cost, duration) |

No new backend events are needed. Rich rendering can use `rich` (add to a `cli` extra:
`pip install "phlox[cli]"`) with a plain-text fallback.

### Approvals in the terminal

This is where reusing the harness pays off. When the gate returns `ask`, the harness
persists a `PendingApproval` and yields `approval_request` + `paused`
(`agent/harness.py:_pause`), and `resume(state, decisions)` continues the turn — designed
so approvals survive disconnects. The CLI simply:

1. renders the pending calls (`run_shell: rm -rf dist && npm run build`),
2. prompts `y` / `n` / `a` (allow + auto-approve the rest of the session),
3. deletes the `PendingApproval` row and calls `session.resume(state, decisions)`.

`PermissionGate(interactive=True)` — a human is present, unlike sub-agents. Tool prefs
(`ToolPref`) carry over, so a user who sets `edit_file: auto` in `~/.phlox` keeps that
policy in every project. `--auto-approve` maps to the existing `auto_approve` flag, same
as the web UI's Agent mode toggle.

### Ctrl-C = the existing cancel path

First Ctrl-C sets the session's `cancel_event` — the same event the chat router sets on
client disconnect — which kills in-flight process trees and stops the loop cleanly
(`sandbox/runner.py:_kill_process_tree`). Second Ctrl-C exits the REPL.

### Slash commands (v1 set)

`/diff` (shell out to the project's git), `/undo` + `/checkpoints` (shadow-repo restore,
Part 1), `/model <name>`, `/tools` (list + toggle prefs), `/skills` (the Skills feature is
already slash-command shaped in the composer — mirror it), `/quit`.

### Packaging

Phase A does the packaging work this rides on: ship `app` as an installable package with
the built frontend embedded (`phlox serve` needs it; `phlox agent` doesn't). `uv build`
already produces a wheel from `backend/pyproject.toml`; the additions are the
`[project.scripts]` entry, a `cli` extra for `rich`, and a CI job that builds + smoke-runs
`phlox --help`.

---

## Rollout

| Phase | PR | Depends on |
|---|---|---|
| A | Ops CLI: `phlox serve/init/admin/rag` + packaging | — |
| B0 | AgentCore `sync_ignore` + `respect_gitignore` | — (independently useful) |
| B1 | BYOW: `workspace_root` column, resolver, allowlist, shadow checkpoints | — |
| B2 | `phlox agent` REPL | A, B1 |
| C | `phlox agent --remote <url>`: client over `POST /v1/agent/completions` | Gateway Phase 2 |

Each phase is a self-contained PR with tests and a doc page, per upstream house style.
Propose the whole design as an upstream issue first (this document is written to be
attachable to that issue), flagging the two decisions most likely to need maintainer
input: the shadow-checkpoint approach vs. disabling checkpoints for BYOW, and the
global-vs-per-user allowlist.

## Testing

- **BYOW:** unit tests for allowlist enforcement (symlink escape, `DATA_DIR` rejection,
  config-removal lockout), traversal guard re-rooting, and shadow checkpoints leaving the
  project's `.git` untouched (assert `git -C project log` is unchanged after a
  checkpoint+restore cycle).
- **Sync ignore:** extend `scripts/e2e_agentcore.py` to plant `node_modules/` + `.git/`
  and assert they never reach the session nor sync back.
- **CLI:** drive `phlox agent` with the existing scripted-provider test harness (the
  pattern `backend/tests` already uses for agent-loop tests) — scripted tool calls,
  scripted approval, assert rendered output and that `resume` ran; no live model needed.

## Open questions

1. Shadow checkpoints vs. checkpoint-off for BYOW (proposal: shadow repo).
2. Should the web UI expose BYOW at all in v1, or keep it CLI/API-only until per-user
   allowlists exist? (proposal: API + CLI only; UI later.)
3. Does `phlox agent` warrant its own default tool-permission seed (e.g. `read_file`,
   `glob`, `grep` auto; everything mutating ask) distinct from the web defaults?
   (proposal: yes, seeded once into the `~/.phlox` ToolPrefs on first run.)
