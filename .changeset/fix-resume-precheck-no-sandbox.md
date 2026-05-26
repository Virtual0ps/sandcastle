---
"@ai-hero/sandcastle": patch
---

Fix `resumeSession` precheck false-negative for the no-sandbox provider. When running on the host with no sandbox, the agent writes its session in place under a cwd-derived directory that Sandcastle was reconstructing from the host repo path — missing the worktree path, symlink-resolved paths (e.g. macOS `/tmp` → `/private/tmp`), and the agent's `.`→`-` encoding. The precheck now locates the session by its unique id via a new `findByIdOnHost` capability on `AgentSessionStorage`, so no-sandbox resume works regardless of how the agent encodes its cwd. Sandboxed (docker/podman) runs are unchanged.
