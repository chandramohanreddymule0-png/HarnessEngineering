\
# Reflection Brief — Harness Engineering Capstone
**Name:** Mule chandramohan reddy
**Date:** 2026-10-03

> Each answer below cites an artifact from the project runs by run ID, file path, test count, or captured output. Where an exact value was not present in the captured artifacts, that limitation is stated rather than inferred.

## Environment
- **Model(s):** Claude Haiku 4.5 (observed in the System 1 run summary; the other systems' model IDs were not captured in the reflection artifacts).
- **OS / Python:** Python 3.13 is evidenced by the `cpython-313` artifacts; the OS distribution was not explicitly captured.
- **Approx. API spend:** System 1 recorded an estimated **$0.1146** total for the eight-claim run. A complete project-wide dollar total was not captured in the evidence.

---
## Part 1 — Per-system

### System 1 — Agentic loop

**1. Loop control.**  
In `runs/20261003_054046/traces/claim_02_stolen_bike.jsonl`, the `stop_reason` sequence is **`tool_use → tool_use → tool_use → end_turn`**. The `run()` function in `claims_intake/loop.py` decides the control flow: it returns when `stop_reason == "end_turn"`, executes tool calls and continues when `stop_reason == "tool_use"`, and raises `UnexpectedStopReason` for any other value. The file also states that string-membership checks are not used for control flow and that there is no primary integer-literal iteration cap; `Budget` is the safety mechanism.

**2. Anti-pattern.**  
One anti-pattern checked by `tests/test_antipatterns.py` is using string-membership tests against assistant text to drive loop control. The same audit also checks that the loop does not use a hard-coded integer iteration cap and that `stop_reason` is the loop-breaking signal. In my System 1 evidence, the test suite passed **29 tests**, so the implementation met those structural constraints rather than only succeeding on one example trace.

**3. Tool design.**  
`classify_claim` and `assess_severity` both take a rationale, but their descriptions clearly separate their jobs: classification commits to a claim type and confidence, while severity commits to a low/medium/high bucket after classification. Their schemas also use separate categorical fields and enums, reducing ambiguity in tool selection. Tool failures use structured fields such as `is_error`, `error_category`, `is_retryable`, and `message`, so the agent can distinguish a permanent input/state problem from a retryable failure instead of receiving only an opaque string.

**4. Your numbers.**  
For `claim_02_stolen_bike`, the trace shows **4 turns** and the run summary records an estimated cost of **$0.0177**. The same run summary reports the total eight-claim spend as **$0.1146**. The README sample value is not present in my captured evidence, so I am not making up a numeric difference between the sample and my run.

### System 2 — Context strategy

**5. Reduction.**  
`runs/20261003-054918/budget.json` records **38,708 baseline tokens**, **16,856 assembled tokens**, and a **56.45% reduction**. The `active` section dominates the assembled context at **15,789 tokens**, versus 204 for `case_facts`, 435 for `resolved_refund`, and 446 for `resolved_subscription`. It remains verbatim because it contains the current unresolved troubleshooting state, including exact information such as the structured payment-update status required by Q6.

**6. Summarize vs preserve.**  
The strategy compresses information that is already resolved while preserving the active conversation and compact structured facts needed for exact retrieval. The final per-section token counts were **204 (`case_facts`)**, **435 (`resolved_refund`)**, **446 (`resolved_subscription`)**, and **15,789 (`active`)**. The control experiment demonstrates why exact structured facts cannot simply be replaced by conversational prose: Q6 passed with the facts block and failed without it.

**7. Facts block.**  
In `eval.jsonl`, Q6 passed because the model returned the exact token **`in_progress`** from the case record. In `eval_control.jsonl`, Q6 returned **`unknown`** and failed when the structured facts block was removed, while Q1 still passed. This shows that the facts block is not redundant: some values can be recovered from conversational context, but exact structured state may disappear without a dedicated facts representation.

### System 3 — Claude Code config

**8. Path-scoped rules.**  
The React rule has YAML frontmatter with `paths` set to **`src/components/**/*`** and **`src/pages/**/*`** in `.claude/rules/react.md`. This makes the convention activate when Claude edits matching files instead of loading the same React-specific guidance everywhere. The project `CLAUDE.md` explicitly prefers `.claude/rules/` with `paths:` globs for cross-cutting conventions.

**9. Forked skill.**  
`.claude/skills/deploy-check/SKILL.md` declares **`context: fork`** and an `allowed-tools` list containing read/search operations plus narrowly scoped Git/GitHub commands. Running it forked keeps verbose file enumeration and inspection output out of the main session, while the read-only allowlist prevents the skill from modifying files, pushing, or deploying. Without those boundaries, the task would either contaminate the main context with intermediate discovery output or have a larger action surface than the validation job requires.

**10. Scope.**  
`validator_output.txt` contains **`OK`**. A project-level example is **`./CLAUDE.md`**, which the configuration defines as the shared repository entry point stored in Git for the team; `.claude/rules/` is also project-level. A user-level example is **`~/.claude/CLAUDE.md`**, which the project documentation describes as personal, not shared through version control.

### System 4 — Orchestration

**11. Push work down.**  
The shift run reported **0 new defects**, while the resulting summary identified **3 high + 2 medium defects on capacitor-bank-C-7** and **1 low VP-4 vent squeal**. In `shift_monitor/warm.py`, the database defines the index **`idx_defects_shift_ts` on `(shift, ts)`** and includes the incremental query `SELECT * FROM defects WHERE ts > ? ORDER BY ts DESC LIMIT ?`. The exact warm-tier total row count was not captured in my terminal evidence, so I do not assert a number for it; the architectural point is that historical records remain in the warm store and only the relevant incremental slice is exposed to the model.

**12. Crash recovery.**  
`shift_monitor/recovery.py` defines **`STALE_RESUME_THRESHOLD_MINUTES = 30`**; a recent partial step is resumed when it is within that threshold, while a stale partial causes a **fresh** start. A fresh restart can be more reliable when the previous partial state is stale because the system avoids continuing from potentially outdated intermediate reasoning while still injecting the durable summary needed for continuity. The Shift C scratchpad shows that behavior by carrying forward a concise conclusion even when the run found **0 new defects**.

**13. Small state.**  
`data/hot_state.json` was **643 bytes** in the captured run. Small persistent state limits context and storage growth as shift runs continue indefinitely. The scratchpad similarly stores compact hypotheses and conclusions instead of copying the full historical defect dataset into every run.

---
## Part 2 — Synthesis

**14. Three layers.**  
The **Model** layer is visible in System 1 traces, where Claude emits tool calls and `stop_reason` values. The **Harness** layer is visible in `claims_intake/loop.py`, which deterministically interprets those values and enforces the loop contract. The **Orchestration** layer is visible in System 4's warm database, hot state, and scratchpad, which manage incremental retrieval and continuity between shift executions.

**15. Deterministic vs prompt.**  
Deterministic code is appropriate for invariants that must not be violated, such as stop conditions, tool preconditions, error formats, and state transitions. Prompt guidance is appropriate for judgment-heavy behavior such as deciding which facts to gather, whether a claim is genuinely ambiguous, or when clarification is useful. In the project, the model makes those semantic decisions while the harness constrains what actions are allowed and how failures are handled.

**16. Context two faces.**  
System 2 handles **intra-session context pressure**, reducing **38,708 tokens to 16,856** for a 56.45% reduction. System 4 handles **cross-session continuity** with a **643-byte** hot state and compact scratchpad entries. The two designs solve different problems: one compresses a conversation without losing critical exact facts, while the other persists only the minimum state needed to continue later.

**17. Reliability you can't see in one run.**  
A single successful trace cannot prove that an implementation is structurally safe against brittle control logic. The System 1 anti-pattern tests specifically verify that assistant-text membership and hard-coded integer iteration caps are not being used to stop the loop. That matters because such an implementation could pass one fixture and still fail when the model's wording or tool-use sequence changes.

**18. Blast radius.**  
For System 1, the operational boundary is the current claim session, so a bad model step is constrained by the session and tool interface rather than directly changing arbitrary application state. The `Budget` is the main safety/kill-switch mechanism, while terminal-tool preconditions and structured tool errors provide enforcement around model actions. Persisted routing/escalation records and session state keep the durable state explicit rather than hidden inside the model's prose.

---
## Part 3 — Honest assessment

**19. What broke.**  
The clearest failure in the final evidence was the intentional System 2 control experiment: **Q6 failed** when the structured facts block was removed, returning `unknown` instead of `in_progress`. That failure was useful because it demonstrated a real dependency in the design rather than hiding it behind a single successful run. The final main evaluation passed Q1–Q6, and System 3's validator returned `OK`, so I did not encounter a major unresolved implementation failure in the final runs.

**20. What change architecture?**  
The biggest architectural shift was separating model judgment from deterministic enforcement and durable state. System 1 moved loop control to `stop_reason` and explicit budget/error handling, System 2 separated summarized history from exact structured facts, and System 4 separated warm historical data from tiny persistent hot state and scratchpad summaries. This made the overall system less dependent on the model behaving correctly by default and gave each responsibility a clearer boundary.

---
## Evidence references

- **System 1:** `runs/20261003_054046/summary.md`; `runs/20261003_054046/traces/claim_02_stolen_bike.jsonl`; `pytest_S1.log`
- **System 2:** `runs/20261003-054918/budget.json`; `runs/20261003-054918/eval.jsonl`; `runs/20261003-054918/eval_control.jsonl`; `pytest_S2.log`
- **System 3:** `validator_output.txt`; `evidence_rule_react.md`; `evidence_rule_skill.md`; `evidence_rule_claude.md`; `pytest_S3.log`
- **System 4:** `shift_run_output.txt`; `hot_state_size.txt`; `scratchpad_line.jsonl`; `shift_monitor/warm.py`; `shift_monitor/recovery.py`; `pytest_S4.log`
