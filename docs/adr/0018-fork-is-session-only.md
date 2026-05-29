# `.fork()` isolates the agent session, not the branch or sandbox

## Context

Per [ADR 0011](./0011-resume-is-one-iteration.md), `RunResult.resume(prompt, options?)`
runs exactly one iteration that continues the last captured **agent session**.
Resume _mutates_ that session — the agent appends to the existing JSONL and reuses
its id — which is the wrong shape for fan-out, where a single parent run is the
starting point for several independent children. `Promise.all([r.resume(a),
r.resume(b)])` against the same `r` collides on the parent JSONL: both children
race to write the same file and the surviving session is undefined.

Both **agent providers** that ship `sessionStorage` today expose a native
fork primitive: Claude Code's `claude --resume <id> --fork-session` and Codex's
`codex exec fork <id>`. Both leave the parent JSONL intact and write the
continuation under a fresh session id.

## Decision

Add `RunResult.fork(prompt, options?)` as a sibling of `.resume()`. It runs
exactly one iteration that continues from the last captured session, but the
parent JSONL is left untouched and the child gets a new session id. The shape
deliberately mirrors `.resume()`:

- **Same options type.** Both take `ResumeRunResultOptions`, which already
  permits `branchStrategy` — the knob fan-out callers actually need.
- **Same single-iteration constraint** ([ADR 0011](./0011-resume-is-one-iteration.md)).
- **Same capability gating.** Both are optional methods that runtime-throw when
  no `sessionId` was captured. No `supportsFork` field, no conditional `never`
  typing — fork should not be more strictly typed than its resume sibling.
- **No `RunResult` marker.** The child's differing `sessionId` is the only
  observable difference and is sufficient.

Internally, `forkSession?: boolean` threads
`RunResult.fork() → run() → orchestrate() → AgentProvider.buildPrintCommand`:
Claude appends `--fork-session` alongside `--resume`; Codex swaps the verb
from `codex exec resume` to `codex exec fork`. `forkSession` is exposed on
`RunOptions` only as an `@internal` field — callers reach it via `.fork()`,
not by setting the flag directly.

### Provider scope

Both providers with `sessionStorage` (Claude and Codex) are wired. Gating
fork on `sessionStorage` and only wiring Claude would silently degrade
Codex `.fork()` into a parent-mutating `codex exec resume`, which is
exactly the trap `.fork()` exists to avoid.

## Considered Options

1. **`fork: true` flag on `.resume()`** — rejected. The capability difference
   (mutates parent vs. doesn't) is large enough that two siblings read more
   clearly than an overloaded `resume`, and fan-out call sites become greppable.
2. **`supportsFork` field with conditional `never` typing** — rejected. ADR 0012
   originally proposed a `supportsResume` capability and the conditional `never`
   shape was never built. `.resume()` ships today as an optional method that
   runtime-throws; `.fork()` follows that same pattern rather than introducing a
   stricter shape for the newer sibling.
3. **Auto-isolate branches and serialize merges inside `.fork()`** — deferred.
   Real concurrent fan-out also needs unique branch names and non-racing merges,
   but those concerns are independent of session forking and have their own
   design space. See "Consequences" below.

## Consequences

- **Fork is session-only.** `--fork-session` and `codex exec fork` isolate the
  agent session JSONL. They do **not** isolate the branch, the worktree, or the
  sandbox. Safe concurrent fan-out (`Promise.all([r.fork(a), r.fork(b)])`)
  requires the caller to give each child a distinct `branch` via
  `branchStrategy: { type: "branch", branch: "..." }`. The default `head` and
  `merge-to-head` strategies are **not** safe for concurrent forks:
  - `head` shares the host working directory across all children.
  - `merge-to-head` creates a temp branch per child and merges back to host
    HEAD; concurrent `git merge` into the same HEAD races the index.
- **`generateTempBranchName` now appends a 6-hex-char random suffix.** The
  pre-fork format `sandcastle/<YYYYMMDD-HHMMSS>` is second-granularity and
  collides under any concurrent invocation, not just fork — that was a latent
  bug for plain `Promise.all([run(), run()])` too.
- **Codex round-trip is a verification gate before merge.** Claude's
  `--fork-session` has been confirmed end-to-end via host↔sandbox JSONL
  transfer; `codex exec fork <id>` has only been confirmed via the CLI surface
  and needs the equivalent round-trip test before this ships.
- **Full auto-isolation is deferred.** Issue-track a follow-up to make
  `RunResult.fork()` either (a) auto-pick unique branches per call or (b)
  refuse `head`/`merge-to-head` at call time so the failure is loud rather
  than silent. Either path needs its own design.

## See also

- [ADR 0011 — `.resume()` is exactly one iteration](./0011-resume-is-one-iteration.md)
- [ADR 0012 — Agent providers own their session storage end-to-end](./0012-agent-provider-owned-session-storage.md)
- [ADR 0016 — Resume support requires filesystem-backed session storage](./0016-resume-requires-filesystem-backed-sessions.md)
