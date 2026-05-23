# Hermes Guardrail Implementation Plan (Pattern 6 – Safe Autonomy with Guardrails)

**Source:** Derived from ideation log `hermes-ideation-pattern6-safe-autonomy.md` (2026-05-19).

## Overview
Implement a permission‑engine that allows constructive agent actions while blocking harmful ones, enabling safe autonomous operation within explicit allow/deny boundaries.

## Policy Schema (YAML/JSON)
```yaml
permissions:
  allow:
    - Read
    - Glob
    - Grep
    - LS
    - Edit
    - Bash(dev test *)
    - Bash(dev style *)
    - Bash(git status)
    - Bash(git diff *)
    - Bash(git add *)
    - Bash(git commit *)
  deny:
    - Read(**/.env*)
    - Bash(git push *)
    - Bash(dev deploy *)
    - Bash(bin/rails db:drop *)
    - Bash(rm -rf *)
  defaultMode: acceptEdits   # or "reject" for deny‑by‑default
```

## Phase‑by‑Phase Roadmap

### Phase 0 – Pre‑Flight (Day 0)
1. Collect source ideation doc and locate Hermes action dispatcher.
2. Audit existing ad‑hoc permission checks (grep for `allow`, `deny`, `permission`, `sudo`, `env`).
3. Document current gaps (e.g., ability to read `.env`, push git, run destructive bash).

### Phase 1 – Design Guardrail Schema (Week 1‑2)
4. Formalize policy structure matching the schema above.
5. Choose enforcement mechanism:
   - **Option A (lightweight)**: Middleware wrapper around `hermes_tools` – each tool call passes through `guardrail.check(action, args)`.
   - **Option B (formal)**: Integrate Open Policy Agent (OPA) or Casbin as a sidecar; agent queries before execution.
   - **Recommendation**: Start with Option A for speed; keep Option B as future upgrade.
6. Produce design document (`guardrail-design.md`) with schema, threat model, failure‑mode analysis, and sequence diagram.

### Phase 2 – Build & Integrate (Week 3‑5)
7. Create guardrail module (`~/hermes/scripts/guardrail.py`):
   - `load_policy(path)` → parsed allow/deny lists (fnmatch‑style globs).
   - `check(action, args)` → returns `(bool, reason)`.
   - Decorator `@guarded` to wrap existing tool functions (`read_file`, `write_file`, `terminal`, etc.).
8. Hook into Hermes agent dispatcher:
   - Locate central `execute_tool(name, *args)`.
   - Insert `if not guardrail.check(name, args): raise PermissionError(...)` before side‑effects.
9. Wire configuration:
   - Add `guardrail_policy_path` to `~/.hermes/config.yaml` (default: `~/.hermes/guardrail/policy.yaml`).
   - On agent start, load policy and log “Guardrails loaded: X allow, Y deny”.

### Phase 3 – Test Suite & Validation (Week 5‑6)
10. Write exhaustive pytest cases:
    - **Allow**: `Read('wiki/foo.md')`, `Bash('dev test ./script.sh')`, `Edit('~/hermes/skills/foo/SKILL.md')` → expect `True`.
    - **Deny**: `Read('~/.hermes/.env')`, `Bash('git push origin main')`, `Bash('rm -rf /tmp/xyz')` → expect `False` + log denial.
    - **Edge**: case‑sensitivity, nested paths, symlinks (realpath check).
11. Run test suite in disposable container/VM (no host side‑effects); verify 100 % pass, >90 % line coverage.
12. Dry‑run integration test:
    - Spin up short‑lived Hermes agent with guardrail enabled.
    - Attempt allowed/denied actions via `hermes chat`; confirm denials return clear error and are logged.

### Phase 4 – Audit Logging & Notifications (Week 6‑7)
13. Extend `guardrail.check` to emit structured events:
    - JSON lines to `~/hermes/logs/guardrail.log` (timestamp, user, action, args, decision, reason).
    - Optional push to local Loki or simple file‑tailer for alerting (e.g., Telegram notify on >3 denies in 5 min).
14. Configure log rotation (size 10 MB, keep 7 days) or use Hermes internal log‑pruning.

### Phase 5 – Refine, UI & Documentation (Week 8‑10)
15. Gather feedback: run agent on low‑risk tasks (news digest, light control); note false‑positives/negatives.
16. Iterate policy: adjust globs, add missing Bash prefixes (e.g., `Bash(dev lint *)`, `Bash(dev format *)`); consider warn‑only mode for new rules.
17. Create lightweight UI (optional):
    - `hermes guardrail show` – print current policy.
    - `hermes guardrail test <action> <args>` – simulate decision.
    - `hermes guardrail reload` – pick up yaml changes without restart.
18. Final documentation:
    - Update `/home/mataanek/.hermes/wiki/ideation/guardrail-implementation.md` with:
      - Step‑by‑step install/upgrade guide.
      - Troubleshooting FAQ (e.g., “Why did my `git commit` get blocked?”).
      - Reference to original ideation log.

### Phase 6 – Handoff & Kanbán Close‑Out (End of Month 2)
19. Present finished guardrail to Hex & Wux:
    - Demo allowed/denied actions in shared screen.
    - Hand over test suite for future rule‑specific tests.
    - Move related tickets (from foundation‑five‑points kanban) to **Done**.
20. Retrospective:
    - What worked? (policy language, middleware simplicity).
    - What to improve? (performance overhead, UI desire).
    - Capture action items for next iteration (e.g., integrate OPA for enterprise‑scale policies).

## Next Steps (User Decision)
- Review this plan; if approved, execute phases sequentially.
- If not decided, keep as reference for future planning.

---
*This document is purely descriptive; no execution has been performed yet.*