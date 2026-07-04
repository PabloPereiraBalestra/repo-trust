<!-- session-budget-protocol:start -->
## Session budget protocol

### Budget tracking
- Before starting each work block and after finishing it, read ~/.claude/usage_snapshot.json (Windows: %USERPROFILE%\.claude\usage_snapshot.json) and record rate_limits.five_hour.used_percentage in SESSION_STATE.md (session % start/end per block).
- S/M/L are calibration buckets with default estimates S=5, M=12, L=25 session points. Point estimates may be any integer (8, 15, etc.); the bucket only determines which median applies during self-tuning. Defaults are overridden by measured data per the Self-tuning section.
- Go/no-go rule: if (100 - current used_percentage) < estimated cost of next block + 10 buffer, do NOT start it. Leave the repo stable, update SESSION_STATE.md, and report: "No budget for next block. Session resets at <resets_at converted to local time>."
- No block may be estimated above 20 points. L placeholders above the cap may sit in the backlog marked SPLIT, but must be split into sub-blocks ≤ cap before entering any session plan.
- If the snapshot file is missing, stale (its last-write time predates this session's first prompt — when in doubt, treat it as stale), or rate_limits is null (happens right after session start or /clear), say so and fall back to manual checkpoints: stop after each block and wait for my go.

### Work block protocol
- Blocks are atomic: each one ends in a stable state (compiles, tests pass, clean commit). Never leave work half-done.
- Ordered by dependency. Each block tagged [DESIGN] (architecture decisions, needs my review) or [MECHANICAL] (direct implementation of already-decided work), plus S/M/L.
- One block at a time. After each block: commit, update SESSION_STATE.md, apply go/no-go, stop and wait for my go.
- Never expand a block's scope. Newly discovered work goes into SESSION_STATE.md as a pending block.
- If a block is trending over its estimate, warn me before exceeding it, not after.
- [MECHANICAL] blocks are delegated to the `implementer` subagent (pinned to Sonnet in its frontmatter). The orchestrator reads the snapshot before and after the delegation and logs the block itself. [DESIGN] blocks run in the main thread.
- On each resume, after computing the window plan, recommend the main-thread model and effort for this window: sonnet for admin or MECHANICAL-heavy windows, opus at high effort only when the window contains genuinely hard [DESIGN] blocks. State it in one line; the user switches with /model and /effort.
- If the plan source is ambiguous or the named plan file is absent/outdated, identify candidates from the repo (plan files, recent commits, CLAUDE.md references), state the evidence, and confirm the source with the user before planning. Never plan from a file just because a prompt named it.

### Metrics logging
- After completing each block, append one line to budget_log.jsonl (project root, append-only, never edit past lines):
  {"t":"block","ts":"<ISO local>","tag":"<DESIGN|MECHANICAL>","size":"<S|M|L>","model":"<model.display_name from snapshot>","est":<points>,"actual":<end_pct - start_pct>,"start_pct":<n>,"end_pct":<n>,"commit":"<hash>","clean":<true|false>}
  In manual mode (no usable snapshot): model from session context, actual/start_pct/end_pct = null.
  clean=false if the block later needed a revert or fix commit; record by appending {"t":"fix","ref":"<commit>"}, never by editing.
- On every session close (any reason), append:
  {"t":"session_end","ts":"<ISO local>","end_pct":<n|null>,"cut_reason":"<budget_gate|user_cut|limit_hit|work_done>","blocks_done":<n>,"buffer":<n>,"cap":<n>,"mode":"<auto|manual>"}
  limit_hit = the 5h limit interrupted work mid-block. budget_gate = the go/no-go rule stopped us. work_done = backlog empty.

### Self-tuning
- On every resume, before planning: read the last 30 lines of budget_log.jsonl. Recalibrate each (size, model) estimate as the MEDIAN of its last 5 non-null actuals. Fallback order: (size, model) → size-only median → defaults. Write the result to the Cost calibration section of SESSION_STATE.md. Silent, no announcement needed.
- Second-order rules, only once the log has ≥10 block entries with non-null actuals (before that: calibration phase, first-order only):
  - If any of the last 3 sessions ended in limit_hit: set buffer=15 and cap=15. Announce in one line.
  - If the last 3 sessions all ended in budget_gate with end_pct < 75: set cap=12 to force finer granularity. Announce in one line.
  - If the median absolute estimation error of a (size, model) bucket exceeds 40% over its last 5 blocks: flag it in the resume summary and propose how to re-scope that block size. Do NOT silently redefine what S/M/L mean.
- Buffer and cap changes revert to defaults (10/20) after 5 consecutive clean sessions (no limit_hit, end_pct ≥ 80 on budget_gate cuts).
- Never modify this protocol itself. Structural changes get proposed to the user, with log evidence, and applied only on approval.

### On "cortamos"
- Leave repo stable, update SESSION_STATE.md, append the session_end log line, summarize in 3 lines: what got done, next block, session resets_at (or "manual mode" if no snapshot).
<!-- session-budget-protocol:end -->
