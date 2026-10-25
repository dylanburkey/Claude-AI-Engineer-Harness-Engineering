# Evidence — Harness Engineering Capstone

One folder per system, as the capstone README requires. Each `system-N/` holds the `pytest` log plus the first end-to-end run; `system-N/live/` holds the final live pass against the Claude API (Vocareum proxy, `ANTHROPIC_BASE_URL=https://claude.vocareum.com`), including `usage.jsonl` with the model and token usage of every API call.

Environment: macOS (Darwin 25.5.0), Python 3.15.1, one `uv`-managed venv per system. Live-pass spend ≈ $0.36 (41 calls, 301,295 input / 8,773 output tokens).

## Checklist against the capstone README

| System | README expects | Result | Evidence |
|---|---|---|---|
| **1 Claims Intake** | `pytest` 29 passed | **30 passed** (the exercise README, `exercises/03-dynamic-decomposition/README.md:65`, says 30; the extra test is `test_run_backstop.py`) | `system-1/pytest.log` |
| | `--all` processes 8 fixtures; `summary.md`, `traces/`, `queues/*.jsonl`, `escalations.jsonl`; exits 0 | ✅ run `20261025_133325`: 8 processed, exit 0, 5 of 8 expected outcomes, $0.1302 (Haiku 4.5) | `system-1/live/` (`summary.md`, 8 traces, `queues/`, `escalations.jsonl`, `run.log`) |
| | | earlier run `20261025_132917`: 4 of 8 expected outcomes, $0.1215 | `system-1/` (`summary.md`, traces, queues, `escalations.jsonl`, `run.log`) |
| **2 Retail Context** | `pytest` 30 passed | ✅ **30 passed** | `system-2/pytest.log` |
| | `budget.json` ≥ 50% reduction from ~47k baseline | ✅ live run `20261025-133442`: 38,708 → 16,894 (**56.36%**) via `messages.count_tokens`; character-estimate run `20261025-132417`: 47,144 → 20,347 (56.84%) | `system-2/live/budget.json`, `system-2/budget.json` |
| | `eval.jsonl` 6 questions; `eval_control.jsonl` ≥ 1 regression | ✅ **6/6** passed; control: Q1 held, **Q6 regressed** (both runs) | `system-2/live/eval.jsonl`, `system-2/live/eval_control.jsonl`, `run.log` |
| **3 Monorepo Config** | `pytest` 35 passed | ✅ **35 passed** | `system-3/pytest.log` |
| | validator prints `OK`, exits 0 | ✅ `OK` (no API calls) | `system-3/validator.log`, `system-3/live/validator.log` |
| | structure of `.claude/` | rules (`api.md`, `react.md`, `tests.md`), `commands/review.md`, `skills/deploy-check/SKILL.md`, `standards/` | `system-3/claude-tree.txt` |
| **4 Multi-Shift** | `pytest` 33 passed | ✅ **33 passed** | `system-4/pytest.log` |
| | shift summary printed; `hot_state.json` updated, < ~5 KB; scratchpad line appended | ✅ 17 new defects of 40; `hot_state.json` **965 bytes** (recorded) / **834 bytes** (live Sonnet 4.6); scratchpad appended | `system-4/run-shift-since-2026-04-01.log`, `system-4/hot_state.json`, `system-4/live/` |
| | a fork's isolated scratchpad | ✅ forks `H1-lot-dielectric`, `H2-thermal-cycling`; base hot state unchanged; merge 2 → 4 entries | `system-4/fork.log`, `system-4/forks/` |

## Deviations from the README, and why

- **System 4 `--since`.** The README command as written printed `shift C: 0 new defects` (`system-4/run-shift.log`): the default window is "8 hours ago" and the fixture data ends 2026-04-29. Re-run with `--since 2026-04-01T00:00:00Z` returned 17.
- **System 4 live run truncated.** Sonnet hit `max_tokens` (1,024 output tokens, `system-4/live/usage.jsonl`), so the JSON state block was cut off and the summary saved as `"**Report generated:** …"`. Discussed in brief Q15, Q17 and Q20.
- **Vocareum endpoint.** The course key returned `401 invalid x-api-key` against the default endpoint; setting `ANTHROPIC_BASE_URL` fixed it (brief Q19).
- **Live usage capture.** For the live pass, each system's own entry point was run through a small wrapper that logs `messages.create` usage to `usage.jsonl`; system code was not modified.
- **System 1 offline tool errors.** No tool errors occurred in the live traces, so the structured-error paths were exercised offline with the real executor (`system-1/tool-errors-offline.log`, brief Q3).
