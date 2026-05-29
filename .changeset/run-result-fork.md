---
"@ai-hero/sandcastle": patch
---

Add `RunResult.fork(prompt, options?)` as the sibling of `RunResult.resume()` for fan-out workflows. Both run exactly one iteration that continues from the last captured agent session, but `.fork()` leaves the parent session JSONL intact and writes the child under a new session id — the underlying mechanism is `claude --resume <id> --fork-session` for Claude Code and `codex exec fork <id>` for Codex. `fork` is present only on results from providers with `sessionStorage` (Claude Code, Codex).

Fork isolates the agent session only — not the branch, worktree, or sandbox. Safe concurrent fan-out (`Promise.all([r.fork(a), r.fork(b)])`) requires giving each child a distinct branch via `branchStrategy: { type: "branch", branch: "..." }`; the default `head` and `merge-to-head` strategies are not safe for concurrent forks. See ADR 0018 for the design rationale and the fan-out caveat.

Also: `generateTempBranchName` now appends a 6-hex-char random suffix to its `sandcastle/<YYYYMMDD-HHMMSS>` format. The previous second-granularity timestamp collided under any concurrent invocation, not just fork.
