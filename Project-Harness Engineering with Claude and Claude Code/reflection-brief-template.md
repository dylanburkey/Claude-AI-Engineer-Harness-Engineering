# Reflection Brief — Harness Engineering Capstone

**Name:** Dylan Burkey
**Date:** 10/25/2026

Replace each `→` with your answer. **Every answer cites at least one artifact from your own runs** — a run ID, file path, token count, claim outcome, or test count. Uncited answers do not pass. 3–6 sentences each unless noted. Paste short artifact snippets where they help.

**Environment**

- Model(s): `claude-haiku-4-5-20251001` (System 1 claims agent, System 2 compression + evals); `claude-sonnet-4-6` (System 4 live shift); System 3 makes no model calls. All live calls went through the Vocareum proxy (`ANTHROPIC_BASE_URL=https://claude.vocareum.com`), which served exactly the requested model on every call (logged per call in `evidence/system-{1,2,4}/live/usage.jsonl`).
- OS / Python: macOS (Darwin 25.5.0) / Python 3.15.1, one `uv`-managed venv per system
- Approx. API spend: **≈ $0.36** for the final live pass of all four systems (41 calls, 301,295 input / 8,773 output tokens: System 1 $0.130, System 2 $0.208, System 4 $0.020, System 3 $0.00). Including earlier System 1 runs, ≈ $0.50 total.
- Tools Used: Claude Code / Claude
**Test results (all four):** System 1 30 passed · System 2 30 passed · System 3 35 passed · System 4 33 passed (`evidence/system-N/pytest.log`).

---

## Part 1 — Per-system

### System 1 — Agentic loop

1. **Loop control.** Quote the `stop_reason` sequence from one trace. Name the file and function that decides continue-vs-stop, and how.
   → From `runs/20261025_120514/traces/claim_01_kitchen_fire.jsonl` (Haiku 4.5):

   ```
   {"turn": 1, "stop_reason": "tool_use", "tool_calls": [lookup_policy, record_claim_fact ×4], ...}
   {"turn": 2, "stop_reason": "end_turn", "tool_calls": [], ...}
   ```

   The sequence is `tool_use → end_turn` in 2 turns. Turn 1 made five parallel tool calls: `lookup_policy(POL-1001)` plus four `record_claim_fact` calls (incident_date, location, description, injury_party). The decision lives in `claims_intake/loop.py`, function `run()`, as a three-way branch on `response.stop_reason` inside `while True:`.
   - `"tool_use"` (`loop.py:113`): run every `tool_use` block, send all `tool_result`s back as one user message, `continue`. That's turn 1.
   - `"end_turn"` (`loop.py:103`): append the assistant message and return `FinalState`. That's turn 2.
   - Anything else (`loop.py:130`): raise `UnexpectedStopReason`.

   No text is inspected. The structured `stop_reason` is the only loop condition, and `test_stop_reason_is_loop_control` (`tests/test_antipatterns.py:88`) enforces that.

2. **Anti-pattern.** Name one anti-pattern `test_antipatterns.py` checks for. What would break in your run if the loop used it?
   → **Anti-pattern: deciding when to stop by searching the model's text for a keyword.** `test_no_string_membership_against_text_in_loop` (`tests/test_antipatterns.py:28`) parses `loop.py` and flags any `"<string>" in <something>` comparison, such as `if "routed" in text: break`.

   Suppose the loop in my run stopped only when the reply contained "routed" or "escalated". In `claim_01_kitchen_fire` (run `20261025_120514`), turn 2 ended without routing, so the keyword would never match. The loop would keep calling the model on a finished conversation until `budget.check()` raised `BudgetExceeded`, after up to 500k input tokens or 180 s, not 2 turns and $0.0084.

   Worse, `_run_one()` in `claims_intake/run.py` catches `BudgetExceeded` and leaves `state = None`, and the backstop only runs when `state is not None and not session.terminal_called`. So the claim would show `outcome=error`, never reach `escalations.jsonl`, and no human would be asked to review it. The opposite failure is just as bad. A turn-1 reply like "I'll look this up and get it routed" would match the keyword and stop the loop before the five `tool_result`s were sent back.

3. **Tool design.** Pick two tools with overlapping inputs. How do the descriptions prevent misrouting? What did a structured tool error let the agent do that a generic string would not?
   → **The pair: `route_to_adjuster` vs `escalate_to_human`.** Both are TERMINAL tools that close the claim, and both take a claim-type enum (`queue` vs `candidate_claim_types`) plus a summary. At the end of every claim the model chooses between exactly these two.

   **How the descriptions prevent misrouting**
   - **They split on the same number.** The 0.6 confidence threshold appears in all three tools that matter, and the conditions fit together without a gap:
     - `classify_claim`: "If confidence is below 0.6, prefer escalate_to_human" (`tools.py:70`).
     - `route_to_adjuster`: requires confidence of at least 0.6 and severity already assessed (`tools.py:133`).
     - `escalate_to_human`: "below 0.6 even after clarification, or when the claim cannot be routed safely" (`tools.py:152`).
   - **The input shapes match the decision.** A single `queue` field can't express "it's theft or auto". The escalation schema demands `candidate_claim_types[]`, `root_cause`, `confidence` and `recommended_action`.
   - **Code backs up the descriptions.** Routing is refused until `classify_claim` and `assess_severity` have run, and both terminal tools refuse once `session.terminal_called` is set (`tools.py:286`, `tools.py:315`).

   **Two gaps.** Nothing forces the model to call *either* tool. My live run `20261025_133325` hit that on 4 of 8 claims, and `run.py`'s backstop caught each one. Also, `route_to_adjuster` only checks that `queue` is a valid enum value (`tools.py:290`). It never compares it with `session.classification["claim_type"]`, so "classified theft, routed to auto" would succeed.

   **What the structured error made possible.** My live traces contained no tool errors, so I triggered each error path offline with the real executor (`evidence/system-1/tool-errors-offline.log`):

   ```
   route before classify          {"is_error": true, "error_category": "permanent", "is_retryable": false, "message": "classify_claim must be called before routing"}
   lookup typo'd policy           {"is_error": true, "error_category": "permanent", "is_retryable": false, "message": "policy_id 'POL-10O1' not found"}
   route after classify+assess    {"routed": true, "queue": "property_damage"}
   escalate after routing         {"is_error": true, "error_category": "permanent", "is_retryable": false, "message": "terminal tool already called this claim"}
   ```

   A bare `"Error"` would leave the model guessing. The structured version lets it:
   - **Fix the cause and recover.** The message names the missing step. Line 3 shows the same route succeeding once `classify_claim` and `assess_severity` have run.
   - **Know that retrying is pointless.** For the typo'd policy, `is_retryable: false` says the same lookup won't work, so the model should ask the claimant or escalate. The catch-all for unexpected exceptions returns `transient` / `is_retryable: true` (`tools.py:375`).
   - **Stop instead of closing the claim twice.** "terminal tool already called" tells it the claim is already closed.

   The system prompt tells it to use these fields: "Read the message and adapt — do not retry blindly" (`system_prompt.py:41`). One caveat: `loop.py:121` puts the error JSON only in the `tool_result` content. It never sets the API's own `is_error` flag on the block, so the model learns about failures only by reading the JSON.

4. **Your numbers.** Quote the turn count and cost for one claim. How does it differ from the README sample, and why?
   → **`claim_01_kitchen_fire`, run `20261025_120514`:**

   | | Value |
   |---|---|
   | Turns | 2 (`tool_use → end_turn`) |
   | Tokens | 6,266 in / 423 out (turn 1: 2,809 / 337; turn 2: 3,457 / 86) |
   | Cost | $0.0084 (Haiku 4.5 at $1 / $5 per M tokens, `pricing.py`) |
   | Outcome | `escalated` (deterministic backstop) |

   The README sample (`exercises/03-dynamic-decomposition/README.md:65`) expects the full set to show "7 routed, 1 escalated" at "≈ $0.05 on Haiku 4.5", about $0.006 per claim. My full live run `20261025_133325` differs on both counts: **5 of 8 claims matched** (03, 05, 07, 08 routed; 06 escalated as expected, though through the backstop rather than a deliberate `escalate_to_human` call), and the run cost **$0.1302**, about 2.6× the README figure.

   **Why fewer turns.** On the failing claims (01, 02, 04), the model looked up the policy and returned `end_turn` after 2 turns without calling `classify_claim` or a terminal tool, and the backstop escalated them. The failure moves between claims: across my two 8-claim runs, `20261025_132917` failed 01, 03, 04 and 07, while `20261025_133325` failed 01, 02 and 04. So it's a model behaviour problem, not a loop problem.

   **Why higher cost.** Every request resends the system prompt and all seven tool schemas, about 2.8k input tokens before any conversation (turn 1 above). Routed claims run 4–6 turns, with input growing as tool results pile up: `claim_03_water_damage` took 6 turns and 21,694 input tokens, for $0.0275. The README's ≈ $0.05 likely came from an earlier, smaller prompt and tool set.

### System 2 — Context strategy

5. **The reduction.** From `budget.json`: baseline tokens, assembled tokens, reduction %. Which section dominates the assembled context, and why keep it verbatim?
   → From `runs/20261025-133442/budget.json` (live run, `messages.count_tokens` counter):

   | Baseline | Assembled | Reduction |
   |---:|---:|---:|
   | 38,708 | 16,894 | **56.36%** |

   An earlier run with the character-count estimate, `runs/20261025-114039`, reported 47,144 → 20,255 (57.04%). The reduction is nearly identical, but the authoritative token counts are the `count_tokens` ones.

   **The active segment dominates:** 15,789 of 16,894 tokens, or 93.5%. All of the saving comes from the two resolved threads, which shrank from about 22,900 raw tokens to 919 (364 + 555). Nothing was removed from the active section.

   **Why it stays verbatim.** The issue is unresolved, so it's not yet known which details will matter. A resolved thread's important facts have settled, and the summary keeps them: the refund section of `context.md` still holds `$22.14`, `ORD-77310` and claim ref `DM-2026-441872` (lines 25–34). On a live thread, the next decision may depend on any detail of what was already tried or refused. For example, the turn-by-turn record of which card the customer tried (`7782` appears 8 times in the active section, `AVS_MISMATCH` 8 times) exists only verbatim. Placing active at the bottom, directly above the new user turn, also puts it where attention is strongest.

6. **Summarize vs preserve.** State the rule for what gets summarized vs kept byte-exact, citing your per-section token numbers.
   → **The rule depends on resolution status, not on size, age or position.** A segment is summarized if and only if its `status == "resolved"`. The active segment is joined byte-exact from its raw turns, and a structured Case Facts block is extracted from the whole transcript and exempt from compression. It's a hard rule. `compressor.py:38` raises rather than summarize anything that isn't resolved:

   ```python
   if segment.status != "resolved":
       raise ValueError(
           f"Refusing to summarize segment {segment.issue_id!r} with status {segment.status!r}."
           " Only resolved segments are compressed; the active segment is preserved verbatim."
       )
   ```

   The active path never calls a model: `active_text = "\n\n".join(t.render() for t in seg.turns)` (`compressor.py:68`). Per-section tokens from `runs/20261025-133442/budget.json`:

   | Section | Treatment | Tokens | Share |
   |---|---|---:|---:|
   | `case_facts` | extracted, exempt | 204 | 1.2% |
   | `resolved_refund` | summarized | 364 | 2.2% |
   | `resolved_subscription` | summarized | 555 | 3.3% |
   | `active` | byte-exact | 15,789 | 93.5% |
   | **Assembled** | | **16,894** | |
   | Baseline | | 38,708 | |

7. **Facts block.** Compare `eval.jsonl` to `eval_control.jsonl`. Which question regressed, and what does that prove?
   → **Q6 regressed, in both runs where I ran evals.** With Case Facts, the model answered `in_progress` (6/6 passed). With Case Facts stripped, Q6 failed and Q1 held at `$22.14`, because the refund summary kept the amount word for word.

   | Run | Q1 control | Q6 control (Case Facts removed) |
   |---|---|---|
   | `20261025-124844` (CLI fallback) | ✅ `$22.14` | ❌ answered "**Active issue**", lifted from the section header and presented as the token |
   | `20261025-133442` (live API) | ✅ `$22.14` | ❌ "there is no structured status token recorded in the case record" |

   This shows the Case Facts block carries information nothing else in the context has. In `runs/20261025-133442/context.md`, `in_progress` appears exactly once, on line 21 inside Case Facts. The active section describes the state only in ordinary sentences. The two failures are also different: one run made up a plausible wrong token, the other admitted it had none. So without the block, you can't even predict *how* the answer will fail. Q1 holding shows the summarizer sometimes keeps facts; the Case Facts block is what guarantees they survive compression.

### System 3 — Claude Code config

8. **Path-scoped rules.** Quote the glob frontmatter from one rule file. Why is it better than a directory-level CLAUDE.md for cross-cutting conventions?
   → From `.claude/rules/tests.md`:

   ```yaml
   ---
   description: Conventions for test files (co-located *.test.ts and *.test.tsx)
   paths:
     - "**/*.test.tsx"
     - "**/*.test.ts"
   ---
   ```

   Test files aren't in one directory. They sit next to the code they test under `src/components/`, `src/pages/` and `src/api/`, so a directory-level CLAUDE.md can't reach all of them. You'd have to copy the conventions into every directory, where they drift apart, or put them in the root CLAUDE.md, where they load into every session even when no test is touched. The glob loads the rule only when a matching file is edited, keeps one source of truth, and covers new directories automatically through `**/`.

   Globs also stack. `src/components/Cart/Cart.test.tsx` matches both `react.md` (`src/components/**/*`) and `tests.md`, so Claude gets both sets of conventions. That case is pinned by `test_ac_02_06_test_file_matches_react_and_tests` (35/35 tests passed, `evidence/system-3/pytest.log`), and the validator printed `OK` (`evidence/system-3/live/validator.log`).

9. **Forked skill.** Quote the `context: fork` and `allowed-tools` lines. What does running forked + read-only buy you? What breaks without it?
   → From `.claude/skills/deploy-check/SKILL.md`:

   ```yaml
   context: fork
   allowed-tools:
     - Read
     - Grep
     - Glob
     - Bash(git status:*)
     - Bash(git diff:*)
     - Bash(git log:*)
     - Bash(git rev-parse:*)
     - Bash(git ls-files:*)
     - Bash(gh pr view:*)
     - Bash(gh pr checks:*)
   ```

   **What it buys.** The fork keeps the noisy discovery output in a sub-agent: file lists, `git status --porcelain`, diffs and `gh pr view` JSON. Only the fixed-shape `verdict / checks` summary returns to the main session. The read-only allowlist means the check can inspect but can't change anything, so a "pass" describes the tree you actually have. It also makes pre-approving those tools safe, so the check runs without permission prompts.

   **What breaks without it.** Without the fork, thousands of tokens of throwaway output pile into the working conversation. That brings compaction sooner, and stale `git status` output can mislead later turns. Without the allowlist, a failing check is easy to "fix": stash the dirty files, generate the migration. The gate then certifies its own edits, out of sight inside the fork.

   The validator checks both properties: `_check_deploy_check_skill` in `ecommerce_team_config/__main__.py` requires `context: fork` and rejects write-capable tools, and it printed `OK`. Two caveats: `allowed-tools` pre-approves tools rather than blocking all others, and `git diff` / `git log` accept `--output=<file>`, which writes a file.

10. **Scope.** From the validator output: project-level vs user-level scope. Give one example of each from this config.
    → The validator printed `OK` (`evidence/system-3/live/validator.log`). Its scope check, `distinguishes_project_vs_user_scope` (`ecommerce_team_config/claude_md.py:32`), requires the root `CLAUDE.md` to:
    - name both project-level and user-level scope,
    - state that user-level config is not version-controlled,
    - give a user-level example.

    `test_ac_01_05_documents_scope_and_user_level_not_versioned` pins the same check. **Project-level** config is committed and loads for everyone who clones the repo. **User-level** config lives in `~/.claude/`, applies across all your projects, and never reaches teammates.

    - **Project-level example:** `.claude/skills/deploy-check/SKILL.md`, committed so "the whole team gets the same checks."
    - **User-level example:** the personal `~/.claude/skills/deploy-check-strict/` variant described in that same file. It fails "when the diff exceeds 500 lines without a PR description longer than 200 chars" and "will not affect teammates and will not be committed to the repo."

    `CLAUDE.md` names the usual failure: a team rule placed in `~/.claude/CLAUDE.md` by mistake, which you can diagnose with `/memory`.

### System 4 — Orchestration

11. **Push work down.** Defects the SQL query returned vs warm-tier total. Name the indexed query. Why does the model never see the full history?
    → **17 of 40.** The warm tier was seeded with 40 defects from `fixtures/defects.json`. Run with `--since 2026-04-01T00:00:00Z`, the shift printed `shift C: 17 new defects` (`evidence/system-4/run-shift-since-2026-04-01.log` offline, and `evidence/system-4/live/run-shift.log` live). The query is `WarmStore.defects_since` (`shift_monitor/warm.py:75`):

    ```sql
    SELECT * FROM defects WHERE ts > ? ORDER BY ts DESC LIMIT ?
    ```

    It uses `idx_defects_ts ON defects(ts)` (`warm.py:20`), confirmed by the query plan: `SEARCH defects USING INDEX idx_defects_ts (ts>?)`, and pinned by `test_defects_since_uses_index_and_does_not_load_full_table`.

    The model never sees the full history because choosing rows by time is a simple rule, so SQL does it before the prompt exists. `invocation.rich` builds the prompt from just the hot state and these rows, capped at `LIMIT 50`. Older history lives on as a summary in the hot state and as monthly cold-tier reports generated in code. That's why the live Sonnet call used only **1,662 input tokens** (`evidence/system-4/live/usage.jsonl`). Prompt size grows with new defects, not with plant history, and the model's effort goes on judgment rather than filtering.

    (The README's command without `--since` returned `0 new defects`, `evidence/system-4/run-shift.log`. The default window is "8 hours ago", and the fixture data ends on 2026-04-29.)

12. **Crash recovery.** The resume-vs-fresh decision and its staleness threshold (`recovery.py`). Why is a fresh start with an injected summary sometimes more reliable than resuming?
    → `shift_monitor/recovery.py`:

    ```python
    STALE_RESUME_THRESHOLD_MINUTES = 30
    def decide(state, now):
        if not state.steps: return "fresh"
        if state.complete:  return "fresh"
        if now - state.steps[-1].ts <= timedelta(minutes=30): return "resume"
        return "fresh"
    ```

    An empty or completed manifest always starts fresh. An incomplete one resumes only if its last step is at most 30 minutes old, "~1/16 of an 8-hour shift cycle". `test_recovery_decide_truth_table` pins the edges: 29 and 30 minutes resume, 31 starts fresh. It's part of the 33/33 passing (`evidence/system-4/pytest.log`). The step timestamps can be trusted after a crash because `Manifest.append_step` flushes each line to disk before continuing (`manifest.py:34-40`).

    Starting fresh is more reliable once the session is stale. A resumed session assumes the defects, alerts and thresholds are still what they were. After 30+ minutes on a production line they aren't, and the model tends to keep reasoning from the old picture. A fresh start rebuilds from the current hot state plus the indexed query, carrying forward only the earlier conclusions as a short summary, not the half-finished reasoning. Inside the window, resuming wins, and even then `invocation.resumed` adds a "New defects since last partial step" section. One caveat: `decide()` is tested but not yet called from `pipeline.run_shift`, so today a real crash just starts a normal run.

13. **Small state.** Byte size of your `hot_state.json`. Why does the budget matter for a system run once per shift, indefinitely?
    → **965 bytes** on the recorded-response run (`evidence/system-4/hot_state.json`) and **834 bytes** on the live Sonnet run (`evidence/system-4/live/hot_state.json`). The budget is **5,120 bytes** (`HOT_STATE_BYTE_BUDGET`, `shift_monitor/state.py:13`), so the recorded run used 19% of it. The 965-byte state holds 17 defect IDs (capped at `MAX_RECENT_HASHES = 20`), a one-line summary, 3 alerts and 3 threshold statuses.

    Each shift reads the previous shift's output and writes back a new version, in a loop with no end. A state that gains a few hundred bytes per shift, at three shifts a day, becomes megabytes within a year. The budget caps that, so the prompt in shift 1,000 costs about the same as shift 1 and never overflows the context window. The cap also forces the model to drop resolved alerts, since the full history already lives in the warm and cold tiers.

    It's enforced twice: `_trim_to_budget` (`pipeline.py:74`) drops alerts, and `write_atomic` raises if the state is still over budget. A small payload also keeps the write-to-temp, flush, rename sequence fast. One gap: only alerts are trimmed, so an oversized summary would fail the write.

---

## Part 2 — Synthesis

*Graded on connecting two or more systems. Cite a named file/artifact from each.*

14. **Three layers.** Point to a file/artifact for each layer and justify.
    → **Model — System 1, `runs/20261025_133325/traces/`.** The traces record what the model chose on each turn: which tools to call, with what arguments, and when to emit `end_turn`. `claim_03_water_damage` used 6 turns including a clarifying question, while `claim_01_kitchen_fire` stopped after 2. That's the model layer: judgment, and in this run, inconsistent judgment (5 of 8 claims correct).

    → **Harness — System 1, `claims_intake/loop.py` + `run.py`.** The harness controls one model session. It loops only on `stop_reason`, raises on unexpected stop reasons, enforces the token and wall-clock `Budget`, and returns tool errors as `is_error` results. It also caught the model's gaps: `runs/20261025_133325/escalations.jsonl` records `"model ended its turn without a terminal action (deterministic backstop)"` for claims 01, 02, 04 and 06, so no claim was left without a decision. Even claim 06's "correct" escalation came from the backstop, not from the model calling `escalate_to_human`.

    → **Orchestration — System 4, `shift_monitor/pipeline.py` + `recovery.py`.** Orchestration works across sessions. It decides when Claude runs (once per 8-hour shift), what it receives (17 of 40 rows plus a 965-byte hot state), and what persists afterward (atomic hot-state write, scratchpad append, fork and merge in `evidence/system-4/fork.log`). `recovery.decide` handles crashes with a 30-minute threshold.

    **How they connect:** each layer limits the one below it. System 1's harness turns the model's choices into a guaranteed outcome per claim. System 4's orchestration does the same across sessions. Its single `client.complete` call per shift is where a System 1-style harness should sit, and the live run shows what happens without one: the response hit `max_tokens` (1,024 output tokens, `evidence/system-4/live/usage.jsonl`) and nothing checked `stop_reason`.

15. **Deterministic vs prompt.** Cite one behavior guaranteed in code (terminal tool, read-only allowlist, atomic write, byte budget) and one guided by prompt. When is each right?
    → **Guaranteed in code:**
    - System 1's terminal-tool guard: `session.terminal_called` makes a second terminal call fail ("terminal tool already called this claim", `evidence/system-1/tool-errors-offline.log`), and the `run.py` backstop escalates any claim that ends without one.
    - System 4's byte budget: `write_atomic` raises above 5,120 bytes.

    **Guided by prompt:** whether to route or escalate. That's the 0.6 confidence threshold in the tool descriptions (`tools.py:70/133/152`) plus "Pick one" in `system_prompt.py`.

    The live run shows the difference. The prompt-guided choice held on only 5 of 8 claims in run `20261025_133325`, while the code guarantees held on 8 of 8: every claim got exactly one outcome.

    **Use code** when breaking a rule is unsafe, the rule can be stated mechanically, and it must hold every time: one decision per claim, a bounded state size, no writes during a deploy check. **Use prompts** when the decision needs judgment no fixed rule captures, like how ambiguous a claim is. Then back the prompt with a code check on the outcome, as the backstop does.

    System 4 shows the cost of skipping that back-up. The model was *asked* to end with a JSON state block. When the live response was cut off at 1,024 tokens, no code noticed, and the hot state silently kept the old alerts with the summary `"**Report generated:** 2026-04-30 (End of Shift C)"` (`evidence/system-4/live/hot_state.json`).

16. **Context, two faces.** Compare context management in System 2 (intra-session) and System 4 (cross-session) with cited numbers from both. Same principle, different mechanism — how?
    → **Same principle:** code, not the model, decides what enters the context, and the model gets only what the next decision needs, with stable facts kept in a structured, size-limited form.

    | | System 2 (intra-session) | System 4 (cross-session) |
    |---|---|---|
    | What's large | one 48-turn conversation: **38,708** tokens | an ever-growing defect history: **40** rows and counting |
    | What's sent | **16,894** tokens (−56.36%), run `20261025-133442` | **17** rows + **965**-byte hot state; **1,662** input tokens live |
    | Durable facts | Case Facts block, **204** tokens, at the top | `hot_state.json`, budget **5,120** bytes |
    | How it shrinks | summarize resolved segments (LLM), keep active verbatim | indexed SQL `defects_since` + `LIMIT 50` (no LLM) |
    | Proof it matters | Q6 fails without Case Facts (`eval_control.jsonl`) | budget enforced by `write_atomic`; 33/33 tests |

    **Different mechanism:** System 2 compresses *within* one long conversation. Its boundary is semantic (resolved vs active), so it needs a model to summarize, and it accepts losing some detail on resolved threads. System 4 never builds a long conversation at all. Each shift is a fresh, small prompt assembled from tiered storage, with the boundary set by time (`ts > since`). Selection is exact SQL and the archive stays in the database. System 2's Case Facts and System 4's hot state play the same role, a compact record that survives when everything else is cut, but one carries facts *within* a session and the other *across* sessions.

17. **Reliability you can't see in one run.** Name one behavior a test guarantees that a single successful run would not reveal. Why does it matter before shipping?
    → **System 1's `test_loop_raises_on_unexpected_stop_reason`** (`tests/test_loop.py:132`, among the 30/30 passed). None of my System 1 runs ever produced a stop reason other than `tool_use` or `end_turn`, so a run can't show what happens when the model returns `max_tokens`, `refusal` or `pause_turn`. The test proves the loop raises `UnexpectedStopReason` instead of treating a cut-off response as a finished claim.

    System 4 shows why this matters. Its live run did hit `max_tokens` (1,024 output tokens, `evidence/system-4/live/usage.jsonl`). It has no equivalent guard or test, so it exited 0 and printed a normal-looking result while silently corrupting the hot-state summary. A "successful" run hid the bug.

    Run-to-run variation makes the same point. One run is a single sample from a non-deterministic model. My two 8-claim System 1 runs failed *different* claims (`20261025_132917`: 01, 03, 04, 07; `20261025_133325`: 01, 02, 04). Any single claim can look fine once. Tests pin the edge cases (30 vs 31 minutes in `test_recovery_decide_truth_table`, budget overflow, double terminal calls) that you'd otherwise discover in production.

18. **Blast radius.** Pick one system. What's the blast radius if it misbehaves, and what's the kill switch? Ground it in that system's tools, enforcement points, and state.
    → **System 1, the claims intake agent.** Its only side effects are the two terminal tools, which append to local JSONL files: `queues/<type>.jsonl` (`route_to_adjuster`) and `escalations.jsonl` (`escalate_to_human`). In run `20261025_133325` that meant `queues/auto.jsonl` and `queues/property_damage.jsonl`. The other five tools only read or write in-memory session state.

    **Blast radius:** at worst, one claim lands in the wrong adjuster's queue or reaches a human unnecessarily. The wrong-queue case is real: `route_to_adjuster` checks only that `queue` is a valid enum value (`tools.py:290`), never that it matches the classification. It can't pay a claim, edit a policy, or touch other claims, because each run is scoped to one `ClaimSession`. The unnecessary-escalation case already happened: 3 of 8 routable claims went to humans in my live run, which costs reviewer time, not money.

    **Kill switches, in layers:**
    1. **Per claim:** `terminal_called` allows exactly one terminal action (`tools.py:286/315`), and the backstop escalates claims with none.
    2. **Per run:** `Budget` raises `BudgetExceeded` at `--max-input-tokens` (default 500,000) or `--max-wall-clock-s` (default 180), and an unexpected stop reason raises `UnexpectedStopReason`.
    3. **Operational:** stop the process, or revoke the API key. Output is append-only JSONL with one trace per claim, so a bad run can be audited claim by claim and its queue lines removed.

    The missing piece is a code check that `queue == classification.claim_type`. That would shrink the radius from "wrong queue" to "escalated for review".

---

## Part 3 — Honest assessment

19. **What broke.** One thing that failed first try in your environment, and how you fixed it. (If nothing, what you checked to be sure.)
    → **System 1 failed at startup, then with authentication errors.**
    - The first `python -m claims_intake.run --all` died with `RuntimeError: ANTHROPIC_API_KEY is not set`. Unlike System 2, `claims_intake/client.py` has no CLI fallback.
    - With my course key exported, the API returned `401 - authentication_error: invalid x-api-key` (`request_id req_011CgPMHXUGEu7VaimVTB8zM`). The key is a Vocareum `voc-…` key, which Anthropic's endpoint doesn't accept.
    - Setting `ANTHROPIC_BASE_URL=https://claude.vocareum.com` fixed it. The SDK reads that variable automatically, so no code changed. I verified with a one-claim run (`runs/20261025_132903`) before the full `--all` run (`20261025_133325`, $0.1302).

    A second first-try failure: System 4's README command returned `shift C: 0 new defects` (`evidence/system-4/run-shift.log`), because the default `--since` window is "8 hours ago" and the fixtures end on 2026-04-29. Passing `--since 2026-04-01T00:00:00Z` returned the expected 17.

20. **What you'd change.** One architectural decision you'd make differently, grounded in what you observed.
    → **I'd make System 4's model call check its own result, the way System 1's loop does.** `AnthropicClaudeClient.complete` (`shift_monitor/client.py`) returns only text. It drops `stop_reason`, and `max_tokens` is fixed at 1,024. The live Sonnet run used exactly 1,024 output tokens (`evidence/system-4/live/usage.jsonl`), so the report was cut off before its JSON state block.

    `pipeline.run_shift` then fell back silently: it kept the previous alerts and thresholds and stored the summary `"**Report generated:** 2026-04-30 (End of Shift C)"` (`evidence/system-4/live/hot_state.json`, 834 bytes). The run still exited 0 and appended a normal-looking scratchpad entry.

    I'd change three things:
    1. Return `stop_reason` from the client, and treat anything but `end_turn`, or a missing JSON block, as a failed shift. Leave `hot_state.json` untouched and record the failure in the manifest, so `recovery.decide` can retry.
    2. Raise `max_tokens`, or better, ask for the state update through a tool call with a schema, so it's structured output, not a fenced block the prompt merely requests.
    3. Add a test that feeds a truncated recorded response.

    System 1 already shows the pattern working: `stop_reason` drives control flow, unexpected values raise, and a code backstop catches a missing decision. That's how its 4 under-completed claims in run `20261025_133325` (01, 02, 04, 06) still ended in an auditable escalation and didn't corrupt any state.
