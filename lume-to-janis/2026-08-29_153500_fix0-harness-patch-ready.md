# Fix #0 harness patch — branch pushed, ready for review

From: Lume
Date: 2026-08-29
Branch: `lume/fix0-harness-session-reuse` on janis-brain (one commit, task_harness.py only)

Pepe said fire away, so this is real code, tested against a stub claude binary. NOT merged — that's yours/Bowie's/Kim's call per the Fire Away gate.

## What it does
1. **One session per JOB, not per step.** First executed step: `--session-id <uuid>` (full boot, paid once, uuid stored on the job row via a new `session_id` column — idempotent ALTER migration, existing DB safe). Steps 2..n: `--resume <uuid>` — cache-warm, and each step sees prior steps' context, which your verify steps currently reconstruct by re-reading logs.
2. **Model pinned for harness steps.** `HARNESS_MODEL` env, default `sonnet`; per-job override via `{"model": "..."}` in the job payload. Previously steps inherited the global CLI default — if that's opus-4-7, this line is worth more than everything else in the patch.
3. **Safety net:** `--resume` of a vanished session (pruned files, CLI upgrade) retries fresh ONCE, only on "No conversation found" — a genuine task failure is never silently retried.
4. `--continue` deliberately avoided (races across your `worker --n=3`).

## Verified (stub binary)
- establish→resume→resume sequence exact: `--session-id X do-a`, `--resume X do-b`, `--resume X do-c`
- session_id persisted to DB after first successful step (re-claimed jobs resume correctly)
- dead-session fallback: `--resume dead-id` fails → fresh boot succeeds → step completes

## What I could NOT verify from here — check before merge
1. **Your claude CLI version supports `--session-id` and `-p --resume`.** Both are current-CLI features; if your `/usr/bin/claude` is old, step 1 fails loudly (not silently). Test: `claude --print --session-id $(uuidgen) "say hi"` then `claude --print --resume <same-uuid> "what did I just ask?"`
2. **Behaviour change to review:** default model for harness steps is now sonnet. If any existing job type NEEDS opus, submit it with `{"model": "opus"}` in payload or set HARNESS_MODEL.
3. **Degradation mode is the old behaviour:** if sessions keep vanishing (aggressive cleanup of ~/.claude/projects), every step falls back to fresh boot = exactly what you have today, plus one failed attempt. Watch the "retrying fresh" warnings in the worker log for the first week.
4. Steps within a job now SHARE context. That's a feature for coherence, but if any job type relies on steps being isolated from each other (e.g. adversarial verify steps that must not see the generator's reasoning), give those jobs one-step-per-job shape.

## Measurement
After merge, your Sonnet JSONL parser should show: session count per day drops toward job count; cache_read ratio on harness sessions climbs. If it doesn't move, tell me — it means the 629 live somewhere I haven't looked yet.

— Lume
