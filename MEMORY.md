# MEMORY.md — project notes & decisions

Durable context for the `claude-usage-statusbar` extension. Newest-relevant first.

## Key decision: live API, not local log estimation

- **v0.1 (abandoned):** estimated usage by summing weighted tokens from `~/.claude/projects/*/*.jsonl`
  against configurable budgets. It did **not** match Claude Code's `/usage` (observed 215% / 21% vs.
  real 68% / 69%). Root cause: Anthropic computes the real percentages server-side (weighting models
  and messages differently than a raw token sum), and **no local file stores the real percentages**.
- **v0.2 (current):** reads the real numbers from the live usage endpoint. Verified to match `/usage`.

## The usage endpoint (the important find)

```
GET https://api.anthropic.com/api/oauth/usage
Authorization: Bearer <claudeAiOauth.accessToken from ~/.claude/.credentials.json>
anthropic-beta: oauth-2025-04-20
anthropic-version: 2023-06-01
```

Response (relevant fields):
- `five_hour.utilization` → **Session %**, `five_hour.resets_at`
- `seven_day.utilization` → **Weekly %**, `seven_day.resets_at`
- `limits[]` entries with `group: "session" | "weekly"`, `percent`, `severity`, `resets_at` (fallback)
- Also present (often null on this plan): `seven_day_opus`, `seven_day_sonnet`, `extra_usage`, `spend`.

## Verified facts about local Claude state

- `~/.claude/.credentials.json` → `claudeAiOauth`: `accessToken`, `refreshToken`, `expiresAt` (ms),
  `scopes`, `subscriptionType`, `rateLimitTier`.
- `~/.claude.json` stores the rate-limit **tier** only (`oauthAccount.organizationRateLimitTier`), not
  current usage.
- `~/.claude/projects/*/*.jsonl` lines carry `timestamp`, `type`, and (for `type: "assistant"`)
  `message.usage` token counts — but **no** rate-limit fields.
- There is **no** `claude usage` CLI subcommand; `/usage` is a TUI-only command backed by the endpoint above.

## Transient errors vs. real errors (v0.3.1+)

- `refresh()` distinguishes **transient** failures (`http-429`, `network`, `timeout`, `http-5xx`)
  from real ones (`no-token`, `auth`, etc.).
- On a transient failure **while we already have data** (`hasData`), the last good values stay
  visible and `startStale()` greys them with a slow spinner (600ms, `SPINNER` frames) instead of
  flashing an error. A real error calls `showError()`, which now first calls `stopStale()` so the
  stale animation can't overwrite it.
- **Idle poll (v0.4.1):** self-rescheduling `setTimeout` (not a fixed `setInterval`). Each successful
  fetch schedules the next poll at `refreshBaseSeconds` (default 90s) ±`refreshJitterPct` (default
  ±20%) → ~72–108s, floored at 15s. This de-synchronizes clients.
- **Backoff (v0.4.1):** after **any** failure, full-jitter exponential backoff —
  `delay = random(0, min(backoffCapSeconds=300, backoffBaseSeconds=2 · 2^(attempt-1)))`. A 429's
  `Retry-After` is honored as a floor. `backoffUntil` guards manual/activity refreshes during the
  window; a success resets `backoffAttempt`/`backoffUntil`, calls `stopStale()`, and resumes the
  jittered base cadence.
- Every `refresh()` (timer, manual command, or activity-triggered) reschedules the poll, so the idle
  clock resets after any refresh.
- Refreshes are also synced to Claude Code request activity (v0.4.0).

## Environment notes

- This machine has **no standalone `node`**. Use `ELECTRON_RUN_AS_NODE=1 /usr/share/codium/codium`
  (Node v22 bundled with VSCodium) to run/syntax-check JS.
- VSCodium extensions dir: `~/.vscode-oss/extensions/`. Installed copy of this extension lives there
  and is separate from this source repo — re-copy + reload after edits.

## Status

- **v0.4.1** current. Adds jittered idle poll (90s ±20%) and full-jitter exponential backoff after
  any failure (429 Retry-After as floor). See "Transient errors vs. real errors".
- **v0.4.0**: live values verified against `/usage`. Resets-in sand-timer gauge, 429 hardening,
  usage refresh synced to Claude Code request activity, stale-spinner (keeps last good values greyed
  during transient errors).
- Repo was restructured (v0.3.x): **the repo root is now the extension folder** — there is no longer
  a `claude-usage-statusbar/` subfolder.
- After editing, user must **Developer: Reload Window** in VSCodium to load the updated build.
