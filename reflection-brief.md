# Reflection Brief — Harness Engineering Capstone

**Name:** Raj Biswas
**Date:** 2026-09-25

**Environment**

- Model(s): Claude Haiku 4.5 (`claude-haiku-4-5-20251001`) for System 1 live runs; Sonnet 4.6 was also used for one optional System 1 run; System 2 used the configured Claude model for its live evaluation; Systems 3 and 4 were verified through their local test/validation and offline recorded-response workflows.
- OS / Python: Linux workspace / Python 3.13.0
- Approx. API spend: At least $0.6756 USD in recorded System 1 live runs; System 2 also made API calls, but an exact combined dollar total was not captured in the evidence.

---

## Part 1 — Per-system

### System 1 — Agentic loop

1. **Loop control.** Quote the `stop_reason` sequence from one trace. Name the file and function that decides continue-vs-stop, and how.
   → The trace `capstone-evidence/system-1-agentic-loop/run-20260925_084914/claim_03_water_damage-trace.jsonl` shows the sequence `tool_use → tool_use → tool_use → tool_use → tool_use → tool_use → end_turn` across seven turns. The capstone entry point is `run()` in `claims_intake/run.py`, which invokes the agent loop in `claims_intake/loop.py`. The loop itself makes the continue-vs-stop decision. When the response has `stop_reason == "tool_use"`, the requested tools are executed, their results are appended, and the loop continues; when it is `end_turn`, the assistant response is appended and the loop returns the final state. Other stop reasons raise an `UnexpectedStopReason`, while the `Budget` object provides the safety mechanism rather than a fixed turn count.

2. **Anti-pattern.** Name one anti-pattern `test_antipatterns.py` checks for. What would break in your run if the loop used it?
   → `tests/test_antipatterns.py` checks for using an integer-literal iteration cap such as `for _ in range(5)` or `while turn < 5` as the primary stopping mechanism. A fixed cap could terminate a claim before the model has completed clarification, classification, severity assessment, and the terminal routing action. The test instead requires `stop_reason` to drive loop termination while allowing a safety budget. This matters because the seven-turn trace for `claim_03_water_damage` already demonstrates a multi-step flow that should not be constrained by an arbitrary fixed number.

3. **Tool design.** Pick two tools with overlapping inputs. How do the descriptions prevent misrouting? What did a structured tool error let the agent do that a generic string would not?
   → `route_to_adjuster` and `escalate_to_human` both operate on the claim after the necessary facts have been gathered, but their descriptions distinguish their conditions. Routing is for sufficiently confident claims with severity assessed, while escalation is for low confidence, unsafe routing, missing facts, or policy disputes. The tool contract returns structured errors containing `is_error`, `error_category`, `is_retryable`, and `message`, rather than raising an exception. For example, attempting to route before classification can return a structured permanent error stating that `classify_claim` must be called first, allowing the agent to understand the exact failure and choose another action.

4. **Your numbers.** Quote the turn count and cost for one claim. How does it differ from the README sample, and why?
   → The final individual run for `claim_06_low_confidence_escalation` was run `20260925_094546` and used 2 turns, 6,298 input tokens, 485 output tokens, and an estimated cost of $0.0087. Its observed outcome was `incomplete`, because the live model did not reach the expected escalation terminal action. This differs from the completed flow represented by the README sample, where the expected terminal behavior is reached. The implementation itself passed all 29 automated tests; the difference was observed during the live model run.

### System 2 — Context strategy

5. **The reduction.** From `budget.json`: baseline tokens, assembled tokens, reduction %. Which section dominates the assembled context, and why keep it verbatim?
   → `budget.json` records a baseline of 38,708 tokens and an assembled context of 16,866 tokens, a 56.43% reduction. The `active` section dominates the assembled context at 15,789 tokens, compared with 204 tokens for `case_facts`, 409 for `resolved_refund`, and 482 for `resolved_subscription`. The active section is the current unresolved state, so the strategy preserves it verbatim rather than risking loss of detailed current-session information through summarization. Evidence is in `capstone-evidence/system-2-context-strategy/run-20260925-091507/budget.json`.

6. **Summarize vs preserve.** State the rule for what gets summarized vs kept byte-exact, citing your per-section token numbers.
   → The context strategy summarizes resolved conversation segments while preserving the active segment byte-exact. In the recorded run, the resolved refund section was 409 tokens and the resolved subscription section was 482 tokens, while the active section remained 15,789 tokens and the structured case-facts block was 204 tokens. The implementation explicitly prevents summarizing non-resolved content and constructs the active segment from the raw rendered turns. This keeps current unresolved work precise while reducing historical context.

7. **Facts block.** Compare `eval.jsonl` to `eval_control.jsonl`. Which question regressed, and what does that prove?
   → The full-context evaluation in `eval.jsonl` passed all 6 of 6 questions. In the control evaluation without the case-facts block, Q1 unexpectedly still passed, but Q6 failed because the model could not recover the exact structured `in_progress` status from the remaining conversation. This shows that the case-facts block provides durable structured information that is not guaranteed to remain recoverable from compressed conversational context. The evidence is in `capstone-evidence/system-2-context-strategy/run-20260925-091507/eval.jsonl` and `eval_control.jsonl`.

### System 3 — Claude Code config

8. **Path-scoped rules.** Quote the glob frontmatter from one rule file. Why is it better than a directory-level CLAUDE.md for cross-cutting conventions?
   → The `react.md` rule uses the frontmatter paths `src/components/**/*` and `src/pages/**/*`. This makes the React-specific conventions activate only for the files and directories where they are relevant. A directory-level `CLAUDE.md` would apply based on directory hierarchy rather than explicitly targeting these cross-cutting file patterns. The evidence is the path-scoped rule documented in `capstone-evidence/system-3-claude-code-config/config-evidence.txt`.

9. **Forked skill.** Quote the `context: fork` and `allowed-tools` lines. What does running forked + read-only buy you? What breaks without it?
   → `.claude/skills/deploy-check/SKILL.md` specifies `context: fork` and an allowlist containing `Read`, `Grep`, `Glob`, read-only Git commands, and read-only GitHub PR/check commands. Forking keeps verbose discovery output out of the main session, while the read-only allowlist prevents modifications, pushes, deployments, and migrations. Without the fork, discovery output would consume the parent session's context; without the read-only restrictions, a validation skill could have a larger modification blast radius. The configuration is recorded in `capstone-evidence/system-3-claude-code-config/config-evidence.txt`.

10. **Scope.** From the validator output: project-level vs user-level scope. Give one example of each from this config.
    → The validator output for the project is `OK`, and the evidence clearly demonstrates project-level configuration through `CLAUDE.md`, `.claude/commands/review.md`, `.claude/rules/react.md`, `.claude/rules/tests.md`, and `.claude/skills/deploy-check/SKILL.md`. These files are project-scoped because they live inside the repository configuration. The collected configuration evidence does not establish an actual user-level configuration example, so I am not claiming one existed. This is a limitation of the evidence rather than inventing a configuration that was not verified.

### System 4 — Orchestration

11. **Push work down.** Defects the SQL query returned vs warm-tier total. Name the indexed query. Why does the model never see the full history?
    → The warm SQLite database contained 40 defect records, and the Capstone `run-shift` execution reported `shift C: 0 new defects` — 0 new defects out of the 40 warm records for that run. The primary evidence is `capstone-evidence/system-4-quality-monitoring/run/run-output.txt`. The indexed query is `SELECT * FROM defects WHERE ts > ? ORDER BY ts DESC LIMIT ?`, and `sql-filter-evidence.txt` shows that the `idx_defects_ts` index is used. Filtering and limiting happen in SQLite, so the model receives only the relevant slice rather than the full warm history. The optional `LIMIT 5` + `EXPLAIN` probe is supporting evidence of this push-down behavior, not the primary Capstone result.

12. **Crash recovery.** The resume-vs-fresh decision and its staleness threshold (`recovery.py`). Why is a fresh start with an injected summary sometimes more reliable than resuming?
    → `recovery.py` uses a 30-minute staleness threshold. The recorded recovery checks showed 29 minutes → resume, exactly 30 minutes → resume, 31 minutes → fresh, while completed and empty states also start fresh. A fresh start with an injected summary can be more reliable when the previous hot state is stale or incomplete because it avoids continuing from potentially inconsistent partial state. The evidence is in `capstone-evidence/system-4-quality-monitoring/recovery-evidence.txt`.

13. **Small state.** Byte size of your `hot_state.json`. Why does the budget matter for a system run once per shift, indefinitely?
    → The recorded `hot_state.json` was 643 bytes. The implementation also enforces a 5,120-byte hot-state budget. A bounded state size matters for a system that runs once per shift indefinitely because state must remain small and predictable rather than growing without limit across shifts. The warm database retains historical records while hot state contains only the compact working state needed for the next run.

---

## Part 2 — Synthesis

14. **Three layers.** Point to a file/artifact for each layer and justify.
    → Model: `claims_intake/system_prompt.py` defines the domain workflow, including fact gathering, clarification, classification, severity, and terminal action selection.
    → Harness: `claims_intake/loop.py` and `run-20260925_084914/claim_03_water_damage-trace.jsonl` show the execution loop, tool dispatch, and `stop_reason`-driven termination.
    → Orchestration: `shift_monitor/pipeline.py`, the System 4 run output, and the recovery/state artifacts show coordination across warm data, hot state, and shift execution. Together these separate model reasoning, deterministic execution control, and persistent workflow state.

15. **Deterministic vs prompt.** Cite one behavior guaranteed in code (terminal tool, read-only allowlist, atomic write, byte budget) and one guided by prompt. When is each right?
    → A deterministic example is the System 3 read-only allowlist in `deploy-check/SKILL.md`, or the System 4 5,120-byte state budget enforced in code. These guarantees are appropriate when violating the rule could modify data, exceed a resource boundary, or create an unsafe execution path. A prompt-guided example is the System 1 workflow requiring the agent to gather facts, clarify ambiguity, classify the claim, assess severity, and select one terminal action. Prompt guidance is appropriate for domain reasoning and sequencing decisions where the model needs to interpret information, while hard safety and state-integrity boundaries belong in code. The System 4 `fork_for_hypothesis` evidence also shows deterministic filesystem isolation: `capstone-evidence/system-4-quality-monitoring/fork-evidence/` contains separate `lot-quarantine` and `instrument-drift` hypothesis forks with isolated scratchpads, while the base hot state remained unchanged and the results were merged into the main scratchpad.

16. **Context, two faces.** Compare context management in System 2 (intra-session) and System 4 (cross-session) with cited numbers from both. Same principle, different mechanism — how?
    → System 2 reduces intra-session context from 38,708 baseline tokens to 16,866 assembled tokens, a 56.43% reduction, while preserving the 15,789-token active section verbatim. System 4 manages cross-shift state instead: the warm tier held 40 defect records, and the Capstone `run-shift` reported 0 new defects out of those 40 warm records. The hot state was only 643 bytes. Both systems apply the principle of keeping working context bounded and relevant, but System 2 does it through compression and selective assembly while System 4 does it through database filtering, compact hot state, and recovery rules. The indexed query and its supporting `LIMIT 5` + `EXPLAIN` probe are documented separately in `sql-filter-evidence.txt`.

17. **Reliability you can't see in one run.** Name one behavior a test guarantees that a single successful run would not reveal. Why does it matter before shipping?
    → System 1's `tests/test_antipatterns.py` verifies that the loop uses `stop_reason` rather than an integer-literal iteration cap as its primary termination mechanism. A single successful run could complete normally even if the implementation secretly used a fixed turn limit. The automated test therefore checks an architectural property across the implementation, which matters because future claims may require different numbers of turns.

18. **Blast radius.** Pick one system. What's the blast radius if it misbehaves, and what's the kill switch? Ground it in that system's tools, enforcement points, and state.
    → In System 4, uncontrolled state growth or stale state could affect subsequent shift runs and the summaries produced from them. The implementation limits hot state to 5,120 bytes, bounds warm queries with timestamp filtering and `LIMIT`, and uses a 30-minute recovery threshold to choose between resume and fresh state. These enforcement points limit how much stale or oversized state can propagate into the next shift. The recorded 643-byte `hot_state.json` demonstrates the resulting bounded state in the tested run.

---

## Part 3 — Honest assessment

19. **What broke.** One thing that failed first try in your environment, and how you fixed it. (If nothing, what you checked to be sure.)
    → The first System 1 live run failed because `anthropic==0.39.0` was incompatible with the installed `httpx==0.28.1`, producing `TypeError: Client.__init__() got an unexpected keyword argument 'proxies'`. I fixed the environment by installing `httpx==0.27.2`, after which System 1 ran successfully at the environment level and its automated tests passed. System 2 later required `anthropic==0.69.0`, so I restored System 1 to `anthropic==0.39.0` and verified `httpx==0.27.2`. The final System 1 automated test result was 29 passed.

20. **What you'd change.** One architectural decision you'd make differently, grounded in what you observed.
    → I would use a separate Python virtual environment for each capstone system from the beginning. System 1 uses `anthropic==0.39.0` while System 2 uses `anthropic==0.69.0`, and using one shared environment caused dependency version churn during verification. The README itself recommends separate environments because the systems have different dependency requirements. Separate environments would make each system reproducible and reduce the chance that verifying one system changes another system's runtime.
