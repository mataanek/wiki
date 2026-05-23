# Ideation: Foundation Five Points for Agentic Systems (Local PC + Phone)

**Source:** Adapted from user-provided points (ref: tweet-like message) for local PC + phone setup.

**Core Insight:**  
The first few steps of your setup matter more than any model or framework you pick later. Get them right and you never lose your flow. The foundation nobody posts about.

## The Five Points (Local Adaptation)

> 1. **tmux** – persistent sessions. Disconnect, close the laptop, come back, every session exactly where you left it. Agentic work runs long; your terminal has to survive that.
> 2. **A private git repo** – the memory layer across all agents. They pull, work, merge back; the codebase stays alive between sessions. Context that would die in a chat window lives in the repo instead.
> 3. **Script everything from day one** – setup scripts, boring boilerplate automated. If you will do a thing more than twice, it is a script.
> 4. **Context‑aware agent queries** – use a wrapper (like `hermes-ask`) to feed the agent your current directory, git branch, tmux pane, and recent history for more relevant answers.
> 5. **The Habit that Ties It Together:** Ask the AI itself – for config, for errors, for any of it – let the agent do the lifting, then double‑check what it hands you.

## Ways to Implement & Options (Local Focus)

### 1. Persistent Terminal Sessions (tmux)
- **Ways:** Install tmux, use tmux-resurrect, or switch to wezterm/Zellij with session persistence.
- **Options:**
  - tmux + tmux-resurrect (restart survives reboots).
  - tmuxinator for project‑specific session layouts.
  - Wezterm (native tabs, GPU‑accelerated, session save).
- **High‑Level Plan:**
  - Sprint 1: Ensure tmux is installed on the dev machine.
  - Sprint 2: Create a base tmux config with status bar, mouse support, and plugin (resurrect).
  - Sprint 3: Provide a `hermes tmux-session` command to start a standardized session for agentic work.
- **Alternatives:** Using Docker containers with built‑in shells, or using `screen` (less feature‑rich).

### 2. Private Git Repo as Memory Layer
- **Ways:** Host a private Git repo (GitHub private, GitLab, self‑hosted Gitea) that agents clone/pull/push.
- **Options:**
  - GitHub Private Repo (familiar, integrates with Actions).
  - GitLab (CI/CD built‑in).
  - Self‑hosted Gitea on a VPS behind tailnet (full control) – optional for future.
- **High‑Level Plan:**
  - Month 1: Set up a private repo for Hermes agent code, skills, configs (already done, cron‑push active).
  - Month 2: Add Git hooks to enforce skill validation on push.
  - Month 3: Train agents (Hex, Wux) to treat the repo as the source of truth; automate pull on start, push on finish.
- **Alternatives:** Using a distributed database (e.g., Dolt) or IPFS for version‑less storage (less mature for code).

### 3. Script Everything from Day One
- **Ways:** Write bash/zsh scripts, use task runners (Just, Make, Invoke), or configure shell aliases.
- **Options:**
  - `~/.hermes/scripts/hermes/` for agent‑specific scripts (restart, logs, update-skills, doctor wrapper, ask).
  - Shell aliases in `~/.bashrc` or `~/.zshrc` for quick access.
  - Use `justfile` for per‑project commands.
- **High‑Level Plan:**
  - Month 1: Audit repeated manual steps (agent restart, log viewing, skill updates).
  - Month 2: Create/refine scripts in `~/.hermes/scripts/hermes/`:
    - `hermes-restart.sh`
    - `hermes-logs.sh`
    - `hermes-update-skills.sh`
    - `hermes-doctor-wrapper.sh`
    - `hermes-ask.sh`
  - Month 3: Publish the script collection and encourage contributions.
- **Alternatives:** Using configuration management (Ansible, Chef) for node provisioning (overkill for small team).

### 4. Context‑Aware Agent Queries (`hermes-ask`)
- **Ways:** Create a wrapper script that collects context (cwd, git branch, tmux pane, recent history) and invokes the agent via `hermes chat -q -Q`.
- **Options:**
  - The current `hermes-ask.sh` in `~/.hermes/scripts/hermes/`.
  - Enhance to include more context (e.g., agent SOUL.md summaries, recent wiki edits).
- **High‑Level Plan:**
  - Month 1: Ensure the wrapper is in PATH and functional.
  - Month 2: Gather feedback on usefulness and refine context collection.
  - Month 3: Document usage examples and encourage adoption.
- **Alternatives:** Manually copying context into each query (error‑prone and tedious).

### The Habit: Ask the AI Itself
- **Ways:** Integrate the agent into the workflow: let it suggest SSH aliases (if needed), generate scripts, debug config, then review.
- **Options:**
  - Use the agent to generate PR descriptions, commit messages, or troubleshooting steps.
- **High‑Level Plan:**
  - Sprint 1: Implement a simple `hermes ask` that pipes input to the agent and returns response.
  - Sprint 2: Add context‑aware prompts (e.g., "based on current tmux session and git status, suggest next steps").
  - Sprint 3: Encourage the habit via documentation and on‑boarding.
- **Alternatives:** Using a separate chatbot UI (less integrated) or relying solely on manual lookup.

## Related Notes
- Existing ideation file: `home-speaker.md` (voice control project).
- Link to FreshRSS digest workflow as an example of well‑founded automation.

## Next Steps (Local Setup)
1. Review the five points with the team (Hex, Wux) to confirm relevance.
2. Prioritize implementation: refine tmux config, finalize script collection, promote `hermes-ask` habit.
3. Create a foundation checklist derived from these points.
4. Document usage examples in the wiki and share with the team.
