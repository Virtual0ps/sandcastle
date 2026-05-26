---
"@ai-hero/sandcastle": patch
---

Make planner branch names deterministic. The parallel-planner and parallel-planner-with-review templates previously asked the agent to assign a branch name in the format `sandcastle/issue-{id}-{slug}`, where the slug was re-derived on every planning iteration. Because each iteration runs a fresh agent, this produced a different branch each time, forking new branches off HEAD and discarding accumulated progress. The format is now the deterministic `sandcastle/issue-{id}`, so re-planning the same issue resumes the existing branch.
