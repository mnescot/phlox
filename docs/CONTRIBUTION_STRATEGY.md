# Upstream Contribution Strategy

This fork ([mnescot/phlox](https://github.com/mnescot/phlox)) exists to make meaningful
contributions to upstream [robert-mcdermott/phlox](https://github.com/robert-mcdermott/phlox).
This document ranks candidate contributions **from most to least valuable**, where value is
judged on three axes:

- **Upstream demand** — is it already on the roadmap, or does it fill a gap the maintainer
  has named?
- **User impact** — does it widen the audience or unlock a genuinely new way to use Phlox?
- **Feasibility / acceptance risk** — does it follow existing seams, or does it require the
  maintainer to accept new architecture?

Prior art from this fork: the **AgentCore remote sandbox runner** (off-host code execution
in Bedrock AgentCore Code Interpreter microVMs) was contributed upstream and is now the
reference remote `SandboxRunner` implementation. That sets the pattern to repeat: find a
seam upstream already designed for extension, implement cleanly behind it, document, test.

---

## 1. API Gateway Phase 2 — agentic endpoint (`POST /v1/agent/completions`)

**The single most valuable contribution available, because upstream has already asked for
it.** [ROADMAP.md](ROADMAP.md) marks Gateway Phase 1 (raw passthrough) complete and names
Phase 2 as *next*: an authenticated endpoint that exposes the full agentic harness — RAG,
tools, MCP, skills — programmatically, not just raw model passthrough.

- The hard parts are already seamed: per-user API keys, the `UsageLedger` with a
  `parent_request_id` field explicitly reserved for grouping the N model calls of one
  agentic request, and budget enforcement at the gateway choke point (HTTP 402).
- Design decisions to propose in an issue first: streaming shape (SSE events mirroring the
  chat harness vs. OpenAI-style deltas), how tool approvals behave for an unattended API
  caller (probably: ask-tier tools deny, as `spawn_subagent` already does with
  `PermissionGate(interactive=False)`), and whether a request can pin an assistant/skill.
- **Why it's #1:** it is roadmap-sanctioned (lowest acceptance risk), and it is the
  foundation nearly everything else below builds on — the CLI client, IDE integration, and
  external orchestration all become thin clients over this endpoint.

## 2. CLI — installation, operations, and a terminal agent client

Phlox currently has **no CLI at all**: no `[project.scripts]` entry points, and install is
docker-compose or `scripts/dev.sh`. A CLI is the biggest *adoption* lever available, and it
is the path to using Phlox as a coding agent in an IDE terminal (see #3). Propose it in
phases so each PR is small and independently acceptable:

**Phase A — packaging & ops (`pipx install phlox` / `uvx phlox`):**
- `phlox serve` (uvicorn wrapper, frontend build shipped in the wheel and served by
  FastAPI), `phlox init` (scaffold `config.yml` + data dir), `phlox admin create-user`,
  `phlox rag reindex`, `phlox export`.
- Lowers the barrier for the "runs fully local with Ollama/LM Studio" audience — the users
  most likely to try Phlox and least likely to want docker-compose.
- Makes headless administration scriptable (today user management and reindexing are
  UI/API-only), which strengthens the systemd production story in
  [DEPLOYMENT.md](DEPLOYMENT.md).
- Purely additive (`cli.py` + packaging); very low acceptance risk.

**Phase B — terminal agent client (`phlox chat` / `phlox agent`):**
- A REPL in the terminal, in the spirit of Claude Code / Codex CLI / kiro-cli: streaming
  output, tool-call rendering, approval prompts inline.
- Two viable architectures (not mutually exclusive):
  - **Embedded mode** — run the harness in-process on the local machine (the backend is
    plain Python), with the workspace rooted at the current directory. No server needed;
    model calls go straight to the configured provider. This is the shortest path to a
    Claude Code-like experience and the natural companion to #3.
  - **Client mode** — connect to a remote Phlox server via the Phase 2 agentic endpoint,
    inheriting server-side RAG, memory, budgets, and chargeback. Requires #1.

## 3. "Bring your own workspace" — project-directory mounting for coding-agent use

Today every conversation is jailed to a managed scratch directory
(`data/workspaces/<conversation_id>/`, enforced by `resolve_in_workspace`). That is the
right default for a chat product, but it means Phlox **cannot operate in-place on an
existing project** — the core of the coding-agent use case (working on the repo you have
open in VS Code). Concrete, small contributions:

- **Workspace root override** — allow a conversation's workspace to be an existing
  directory instead of a managed scratch dir. Admin-gated and allowlisted in `config.yml`
  (e.g. `workspace.allowed_roots:`), since it deliberately relaxes the isolation
  assumption; the existing traversal guard simply re-roots. Off by default.
- **Sync ignore rules for remote runners** — the AgentCore runner syncs the whole
  workspace into the microVM before each run and back out after. Fine for scratch
  workspaces; pathological for a real repo (`node_modules/`, `.git/`, build output).
  Add `.gitignore`-aware / configurable ignore patterns to the sync (and to artifact
  capture), plus a documented recipe for large repos (clone inside the session instead of
  syncing). This directly extends this fork's own AgentCore contribution.
- Together with the embedded CLI (#2B), this yields the full "open the VS Code terminal,
  run `phlox`, and it works on the project in cwd" experience.

## 4. Stronger agent orchestration

`spawn_subagent` today is deliberately minimal: flat (no recursion), one hardcoded toolset
(`SUBAGENT_TOOLS`), 8 tool rounds, same model as the parent, free-text report scraped from
the child's event stream. Extensions, in order of value:

- **Typed / named sub-agents.** Reuse the custom-assistants model ("persona + toolset +
  capability limits") so `spawn_subagent` can invoke a named agent definition — a cheaper
  model for mechanical work, a scoped toolset per role. The `Assistant` schema's
  `created_by` / `visibility` fields already anticipate user-defined variants.
- **Structured sub-agent output.** Let the parent pass a JSON schema the child's report
  must satisfy, making decomposition reliable instead of prose-parsing.
- **Background / long-running tasks.** Everything is currently bound to one SSE request.
  A detached job model (run, survive disconnect, notify on completion, resume) pairs
  naturally with the persistence upstream already built for approvals
  (`PendingApproval`) and checkpoints — and is what makes big agentic tasks practical.

## 5. Alembic migrations

The roadmap flags the `_ensure_columns` dev-migration shim as needing replacement now that
optional Postgres support has landed, "before real schema migrations show up."
Unglamorous, explicitly wanted, self-contained — an ideal early PR to build maintainer
trust before the larger items above.

## 6. Tier 5 — data governance / PHI readiness

The only remaining roadmap tier: audit logging, secrets out of `config.yml` (env/vault),
PII/PHI policy, provider data-retention controls, encryption at rest. High eventual value
(it gates the sensitive-data deployments Phlox appears aimed at) but deliberately deferred
by upstream, so coordinate before investing — best contributed incrementally (audit-log
seam first, secrets handling second).

## 7. RAG quality upgrades

Two items along seams that already exist:

- **Cross-encoder reranker** behind the `Reranker` interface (the default is a lexical
  reranker the maintainer describes as "cross-encoder-ready").
- **Inline citation highlight-on-click** in the UI — named in the roadmap as a remaining
  polish item for the citations feature.

## 8. Per-API-key budgets and rate limiting

Named in the roadmap as a "candidate follow-up" to the gateway. Small, useful for anyone
running Phlox as a shared gateway, and it composes with #1.

## 9. Native Anthropic provider

Providers are currently Bedrock and OpenAI-compatible. A first-party `anthropic` provider
via the documented [ADDING_A_PROVIDER.md](ADDING_A_PROVIDER.md) seam (streaming, tool use,
vision, prompt caching) broadens provider coverage the same way the AgentCore runner
broadened sandbox coverage.

## 10. Remaining Tier 4 polish

Explicitly listed as open nice-to-haves: full conversation-branch tree, React/JSX live
artifact previews, command palette, voice in/out. Real but lower leverage; good "keep a PR
in flight" items between larger efforts.

---

## Working method

1. **Open an issue/discussion upstream before any large PR.** The roadmap is clearly the
   maintainer's steering document — frame proposals as roadmap additions with the code
   seams they touch, matching the house style.
2. **Start small to build trust** (e.g. #5 Alembic or the #7 reranker), then take on #1.
3. **One concern per PR**, with tests and a doc page — every feature upstream ships with
   both, and PRs that don't will read as unfinished.
4. **Follow the seams**: `SandboxRunner`, `Reranker`, `Provider`, `Tool`, the ledger's
   `parent_request_id`. The AgentCore contribution succeeded because it implemented an
   interface upstream had already designed for extension.
