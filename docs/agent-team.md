# Agent team

To build Mona's Project Pulse dashboard, I'm using a team of four custom agents defined under `.github/agents/`, orchestrated with GitHub Copilot CLI running in a Codespace.

- **Orchestrator** — Model: Claude Opus 4.7 (copilot). Coordinates the Planner, Coder, and Designer agents: breaks the request into phases, assigns non-overlapping file scopes, runs independent work in parallel, and reports the integrated outcome. Defined in `.github/agents/orchestrator.agent.md`.
- **Planner** — Model: Claude Opus 4.7 (copilot). Researches the repository and requirements, then produces an implementation plan with ordered steps, file assignments, dependencies, parallelizable work, edge cases, and validation expectations. Does not write code. Defined in `.github/agents/planner.agent.md`.
- **Coder** — Model: GPT-5.5 (copilot). Implements the dashboard logic and, for Project Pulse, supporting files like `.vscode/launch.json` so the app can be launched with a predictable configuration and working directory. Defined in `.github/agents/coder.agent.md`.
- **Designer** — Model: Gemini 3.1 Pro (copilot). Owns the UI/UX of the dashboard — project cards, status badges, priority treatment, spacing, and responsive layout — using deterministic CSS hooks such as `.dashboard` and `.project-card`. Defined in `.github/agents/designer.agent.md`.

All four agents operate strictly within the file scopes assigned by the Orchestrator and never stage, commit, or push changes — git operations remain under the learner's control via Copilot CLI prompts.
