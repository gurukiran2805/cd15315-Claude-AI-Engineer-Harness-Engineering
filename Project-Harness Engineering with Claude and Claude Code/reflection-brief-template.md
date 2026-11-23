# Reflection Brief — Harness Engineering Capstone

**Name:** Gurukiran
**Date:** 2026-11-23

Replace each `→` with your answer. **Every answer cites at least one artifact from your own runs** — a run ID, file path, token count, claim outcome, or test count. Uncited answers do not pass. 3–6 sentences each unless noted. Paste short artifact snippets where they help.

**Environment**

- Model(s): `claude-haiku-4-5-20251001` for Systems 1 and 2; System 3 used no model; System 4 used a recorded response
- OS / Python: Windows; Python 3.12.10 in the solution virtual environments
- Approx. API spend: $0.1512 for the Claim run in `runs/20261123_212059/`; the README estimates $1–$5 for all four live-API systems, and System 4 was run offline with a recorded response

---

## Part 1 — Per-system

### System 1 — Agentic loop

1. **Loop control.** Quote the `stop_reason` sequence from one trace. Name the file and function that decides continue-vs-stop, and how.
   → In `runs/20261123_212059/traces/claim_01_kitchen_fire.jsonl`, the sequence is `tool_use`, `tool_use`, `end_turn`. The decision is in `claims_intake/loop.py` in `run()`: `tool_use` executes every tool block and continues, `end_turn` returns the final state, and every other value raises `UnexpectedStopReason`. It is a response contract, not text matching or a fixed turn count.

2. **Anti-pattern.** Name one anti-pattern `test_antipatterns.py` checks for. What would break in your run if the loop used it?
   → `tests/test_antipatterns.py` forbids string-membership checks against assistant text as loop control. If the loop required a magic phrase such as `"done" in response.text`, a valid model response or malformed wording could end or continue the wrong claim. The traced run already used a structured `end_turn` stop reason, so that text heuristic would add unnecessary nondeterminism.

3. **Tool design.** Pick two tools with overlapping inputs. How do the descriptions prevent misrouting? What did a structured tool error let the agent do that a generic string would not?
   → `classify_claim` and `assess_severity` both record a commitment, but their descriptions separate the acts: classify the claim type/confidence once after facts; assess severity only after classification. The terminal descriptions on `route_to_adjuster` and `escalate_to_human` further distinguish safe routing from human escalation. `tools.py` returns JSON with `is_error`, `error_category`, `is_retryable`, and `message`; for example, routing before classification returns a permanent structured error, allowing the loop to correct the state instead of treating an opaque string as a successful route.

4. **Your numbers.** Quote the turn count and cost for one claim. How does it differ from the README sample, and why?
   → Claim `claim_03_water_damage` used 8 turns, 29,569 input tokens, 1,258 output tokens, and cost $0.0359 in `runs/20261123_212059/summary.md`; it asked two clarifications. The README does not give a per-claim sample, but it estimates $1–$5 for all four systems while this entire 8-claim run cost $0.1512. Water damage cost more than simple escalations because its trace took the clarification/classification/severity/routing path rather than ending after an immediate escalation.

### System 2 — Context strategy

5. **The reduction.** From `budget.json`: baseline tokens, assembled tokens, reduction %. Which section dominates the assembled context, and why keep it verbatim?
   → `runs/20261123-212428/budget.json` reports 38,708 baseline tokens, 16,834 assembled tokens, and a 56.51% reduction. The `active` section dominates at 15,789 of 16,834 assembled tokens. It is kept because it contains the unresolved payment-method issue and exact facts needed for the next turn; compressing or dropping it would risk losing the current state that is not represented in the resolved-case summaries.

6. **Summarize vs preserve.** State the rule for what gets summarized vs kept byte-exact, citing your per-section token numbers.
   → The per-section accounting in `budget.json` shows the durable `case_facts` at 204 tokens, the resolved refund at 384, and the resolved subscription at 475; the active conversation remains 15,789 tokens. The rule is to keep the small structured facts byte-exact, summarize resolved or closed branches, and retain the active unresolved branch as the working context. This removes old resolved detail while preserving IDs, exact statuses, amounts, and the live issue.

7. **Facts block.** Compare `eval.jsonl` to `eval_control.jsonl`. Which question regressed, and what does that prove?
   → In `runs/20261123-212428/eval.jsonl`, all six questions pass; the control in `eval_control.jsonl` passes Q1 but fails Q6. Q6 is the exact structured status: the full run answers `in_progress`, while the facts-stripped control incorrectly says no structured status is documented. This proves the durable case-facts block is needed to preserve exact, structured state that the surrounding conversation alone does not reliably expose.

### System 3 — Claude Code config

8. **Path-scoped rules.** Quote the glob frontmatter from one rule file. Why is it better than a directory-level CLAUDE.md for cross-cutting conventions?
   → `.claude/rules/react.md` declares `paths: ["src/components/**/*", "src/pages/**/*"]`. It is better than a directory-level `CLAUDE.md` for React conventions that span separate trees and for rules that need to layer, such as test rules applying everywhere with `["**/*.test.tsx", "**/*.test.ts"]` in `tests.md`. Glob scope makes the convention load only for relevant files, regardless of repository depth.

9. **Forked skill.** Quote the `context: fork` and `allowed-tools` lines. What does running forked + read-only buy you? What breaks without it?
   → `.claude/skills/deploy-check/SKILL.md` contains `context: fork` and an `allowed-tools` allowlist consisting of `Read`, `Grep`, `Glob`, and scoped `git`/`gh` read commands. The fork keeps verbose diff and status output out of the main session, while the allowlist prevents mutation or deployment. Without forking, intermediate discovery can contaminate the main context; without the allowlist, a deploy check could accidentally modify files or push code despite its stated read-only contract.

10. **Scope.** From the validator output: project-level vs user-level scope. Give one example of each from this config.
    → The validator prints `OK`, and `CLAUDE.md` documents project-level scope under `./CLAUDE.md`, `.claude/standards/`, and `.claude/rules/`, plus user-level scope under `~/.claude/CLAUDE.md`, `~/.claude/commands/`, and `~/.claude/skills/`. This repository’s `.claude/skills/deploy-check/SKILL.md` is a project example shared with the team. The user-level example is the optional personalized strict deploy skill that does not affect teammates or enter version control.

### System 4 — Orchestration

11. **Push work down.** Defects the SQL query returned vs warm-tier total. Name the indexed query. Why does the model never see the full history?
    → The shift scratchpad records 17 defects analyzed since 2026-04-01, while the warm SQLite tier contains 40 total defects in `data/warm.sqlite`. `WarmStore.defects_since()` uses indexed `SELECT * FROM defects WHERE ts > ? ORDER BY ts DESC LIMIT ?`, backed by `idx_defects_ts` and limited to 50 rows. The model receives only that recent window plus hot state, so full history stays in SQLite and does not consume prompt context.

12. **Crash recovery.** The resume-vs-fresh decision and its staleness threshold (`recovery.py`). Why is a fresh start with an injected summary sometimes more reliable than resuming?
    → `shift_monitor/recovery.py` chooses `resume` only when a manifest has unfinished steps and its latest step is no older than 30 minutes; it chooses `fresh` for no steps, a complete manifest, or stale work. A fresh start can inject the manifest’s captured summary and avoid replaying old partial tool state that may be incomplete or inconsistent. It trades duplicate work for a cleaner, bounded working set.

13. **Small state.** Byte size of your `hot_state.json`. Why does the budget matter for a system run once per shift, indefinitely?
    → `data/hot_state.json` is 965 bytes, below the 5,120-byte hot-state budget enforced in `state.py`. Keeping it small prevents 20 recent hashes, alerts, and summaries from accumulating across shifts. Without a byte cap, state would grow indefinitely and eventually crowd out the newly gathered defects in every future prompt.

---

## Part 2 — Synthesis

*Graded on connecting two or more systems. Cite a named file/artifact from each.*

14. **Three layers.** Point to a file/artifact for each layer and justify.
    → Model: `runs/20261123_212059/traces/claim_01_kitchen_fire.jsonl` shows the model choosing policy lookup and facts before ending; it makes claim decisions.
    → Harness: `claims_intake/loop.py` and `claims_intake/tools.py` define stop_reason control, budgets, structured tool errors, and terminal routing.
    → Orchestration: `shift_monitor/pipeline.py` coordinates hot state, indexed warm retrieval, invocation, atomic state update, and the scratchpad across shifts.

15. **Deterministic vs prompt.** Cite one behavior guaranteed in code (terminal tool, read-only allowlist, atomic write, byte budget) and one guided by prompt. When is each right?
    → Code guarantees that `HotState.write_atomic()` refuses payloads above 5,120 bytes and replaces the file only through a temporary file, as enforced in `shift_monitor/state.py`. By contrast, the claims system prompt tells the model to request one clarification and classify only after enough facts. Code is right for safety and invariants that must never depend on model compliance; prompts are right for flexible judgment that should adapt to the claim or conversation.

16. **Context, two faces.** Compare context management in System 2 (intra-session) and System 4 (cross-session) with cited numbers from both. Same principle, different mechanism — how?
    → System 2 reduced context from 38,708 to 16,834 tokens (56.51%), with 204 for case facts and 15,789 for active content in `budget.json`. System 4 keeps `hot_state.json` at 965 bytes and pushes 17 recent defects from a 40-row warm store into a shift prompt. Both preserve a small active working set and move resolved/history detail outside the model context. System 2 compresses at assembly time within a conversation; System 4 persists bounded state and retrieves a filtered window across shifts.

17. **Reliability you can't see in one run.** Name one behavior a test guarantees that a single successful run would not reveal. Why does it matter before shipping?
    → `test_antipatterns.py` guarantees statically that loop control never uses string membership or an integer-literal iteration cap and that `stop_reason` is the exit contract. A single trace can pass while the loop later diverges on wording or exceed its turn limit in a novel claim. This matters before shipping because it prevents future changes from silently converting deterministic safety rules into model-dependent behavior.

18. **Blast radius.** Pick one system. What's the blast radius if it misbehaves, and what's the kill switch? Ground it in that system's tools, enforcement points, and state.
    → The Claims system could route a claim to the wrong queue or escalate unnecessarily, affecting claim handling and human workload. The blast radius is bounded by `terminal_called`, which prevents repeated terminal calls, and `route_to_adjuster`/`escalate_to_human` validation in `tools.py`. The kill switch is the loop `Budget` plus `_escalate_without_terminal` in `run.py`, which stops or forces a human escalation instead of allowing an endless loop.

---

## Part 3 — Honest assessment

19. **What broke.** One thing that failed first try in your environment, and how you fixed it. (If nothing, what you checked to be sure.)
    → No first-try application failure was preserved in the run artifacts. I checked the actual verification surfaces instead: all Claims tests collected 30, Context tests reported 30 passed, Config tests reported 35 passed with validator `OK`, and Orchestration tests reported 33 passed. I also verified the live run artifacts, the 965-byte hot-state file, and the warm database count rather than relying on README success expectations alone.

20. **What you'd change.** One architectural decision you'd make differently, grounded in what you observed.
    → I would add machine-readable run-level test and cost metadata beside each capstone artifact directory. The existing evidence is clear but spread across pytest output, summary markdown, JSON, and JSONL, so a manifest containing model, timestamp, test counts, API cost, and artifact hashes would make reproducibility and later audit comparison much faster.
