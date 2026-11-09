# AGENTS.md

Instructions for every agent (and human) working in this repository. Read this file and `PLAN.md` before doing anything.

## 1. Prompt log (`.prompts/`) — mandatory

All prompts are stored in the `.prompts/` folder. It contains **everything ever written by anyone in this repo**: every user prompt and every final agent reply, from every person and every session. It is the project's long-term context and must be consulted for history before making decisions.

Rules:

- Store **only** user prompts and final replies. Never store chain-of-thought, reasoning, intermediate tool output or tool calls.
- One `.txt` file per message, never edited after it is written (append-only history).
- Path: `.prompts/<YYYY-MM-DD>/<HHMMSS>Z_<author>_<role>.txt`
  - time is UTC, `<author>` is the human who drove the session (lowercase handle), `<role>` is `user` or `assistant`.
- Each file starts with a header, a blank line, then the message verbatim:

  ```
  author: <handle>
  role: user | assistant
  timestamp: <ISO 8601 UTC>
  session: <session id or "unknown">
  ```

- **Claude Code writes the log automatically** with the hooks in `.claude/settings.json` (`.claude/hooks/prompt_log.py`). Don't also write it by hand in a Claude Code session, or the turn gets logged twice.
- Other agents and tools without these hooks must write the files themselves: the user prompt file when they receive a prompt, the assistant file when they give the final reply. If a turn is skipped, add it later instead of leaving a gap.
- CI checks the format (`.github/scripts/check_prompt_log.py`).
- Never put secrets, tokens or credentials in a prompt file. If a prompt contains one, redact it as `[REDACTED]` and tell the user.

## 2. Infrastructure- and AI-first

When developing in this project everything is infrastructure and AI-first. That means that before developing any features we always ensure the proper infrastructure is in place, an example is that we start the whole project by setting up CI/CD, access controls (both in infra and in application) before doing anything else. This includes things like tracing, telemetry, and logs. Also setting up A2A integrations should be a first-class citizen, meaning that we always implement it as an integrated part of each feature, taking into consideration if there are any security concerns by giving agents access to any given functionality. We also want to keep the application clean, so we always fix a problem before moving on to the next task. No broken-windows.

In practice, a change is not done until:

- It builds, is formatted, and all tests pass in CI.
- It is authorized (deny by default) both in infrastructure and in the application.
- It emits traces, metrics and structured logs that let an agent debug it end to end (frontend → backend → database → infrastructure).
- It is reachable by agents through the agent integration layer (see `PLAN.md`), with the security implications written down in the PR.
- No known warnings, failing checks or TODO-hacks are left behind. If you find a broken window, fix it (or stop and raise it) before continuing.

## 3. Working agreement

- `PLAN.md` is the source of truth for order of work. Do not start a step that is not agreed in `PLAN.md`. Propose changes to the plan instead of silently deviating.
- Ask before making unusual or hard-to-reverse choices. The project owner does not want to be surprised by awkward choices.
- Access is defined as code in the platform repo (`access/people.yaml`). Only infra developers (and their agents) may work on infrastructure, and even they only change it through approved PRs applied by CI. An agent run by an app developer only uses app logs, metrics and traces. It must not try to access Azure, Kubernetes or the database.
- `reference/` is a read-only example of the architectural patterns (Oxpecker, parse-don't-validate, DTO → domain → DBO, permissions resolved from the JWT email). `reference-infra/` is a read-only example of infrastructure patterns. Copy patterns, not code or business logic. Both folders contain third-party company material: they are git-ignored, must never be committed or pushed, and must never be quoted in `.prompts/`.
- **F# code follows [`docs/fsharp-style.md`](docs/fsharp-style.md)**, in every file: input → domain → action → report, no mutation, no `null`, parse-don't-validate. CI enforces what it can.
- No git pre-commit hooks. Every check (format, lint, tests, secret scanning) runs in CI.
