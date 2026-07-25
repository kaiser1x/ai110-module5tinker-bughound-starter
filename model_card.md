# BugHound Mini Model Card (Reflection)

---

## 1) What is this system?

**Name:** BugHound
**Purpose:** Analyze a Python snippet, propose a fix, and run reliability checks before suggesting whether the fix should be auto-applied.
**Intended users:** Students learning agentic workflows and AI reliability concepts; more broadly, engineers who want a lightweight, explainable first-pass code reviewer that never silently trusts an LLM's output.

---

## 2) How does it work?

BugHound runs a 5-step agentic loop in `BugHoundAgent.run()`:

1. **PLAN** — logs intent to scan and propose a fix. Fixed, not adaptive.
2. **ANALYZE** — `analyze()` checks `_can_call_llm()`. If no client (Heuristic mode) or the mode is offline, it runs `_heuristic_analyze()`: regex/substring checks for `print(`, bare `except:`, and `TODO`. If Gemini is available, it sends `prompts/analyzer_system.txt` + `analyzer_user.txt` to the model and expects back a raw JSON array of `{type, severity, msg}` objects. If the API call throws, or the response isn't parseable JSON (`_parse_json_array_of_issues` returns `None`), it falls back to heuristics.
3. **ACT** — `propose_fix()` follows the same heuristic-vs-LLM branch. Heuristic fixer does mechanical regex substitution (bare except → `except Exception as e:`, `print(` → `logging.info(`). LLM fixer sends issues + code to `fixer_system.txt`/`fixer_user.txt`, expects raw Python code back (no fences), strips markdown fences defensively, and falls back to heuristics on API error or empty output.
4. **TEST** — `assess_risk()` (in `reliability/risk_assessor.py`) scores the diff between original and fixed code: issue severity, whether the fix is suspiciously much shorter, whether `return` statements disappeared, whether a bare `except` was touched.
5. **REFLECT** — the agent reads `risk["should_autofix"]` and logs whether the fix would be auto-applied or deferred to a human. It does not actually apply the fix anywhere — this is a recommendation, not an action.

Heuristics are the default, deterministic backbone. Gemini is treated purely as a swappable *tool* for steps 2 and 3 — if it errors, times out, hits rate limits, or returns garbage, the agent transparently drops back to heuristics and logs that fact. Risk assessment and the auto-fix decision are never delegated to the LLM.

---

## 3) Inputs and outputs

**Inputs tested:**
- `flaky_try_except.py` — a function wrapping a bare `except:` around a file read.
- `mixed_issues.py` — TODO comment + print + bare except in one function (multiple simultaneous issues).
- `print_spam.py` — clean control flow but noisy `print()` calls.
- `cleanish.py` — already using logging, no real issues (checks for false positives).

**Outputs observed:**
- Heuristic mode issues: `Code Quality/Low` (print statements), `Reliability/High` (bare except), `Maintainability/Medium` (TODO comments).
- Gemini mode issues on `mixed_issues.py`: 3 findings — `Maintainability/Low` (TODO), `Readability/Low` (print in business logic), `Reliability/High` (bare except, with a much richer explanation: "catches all exceptions, including system exits and keyboard interrupts"). Risk score 45, MEDIUM, no autofix.
- Gemini mode issues on `flaky_try_except.py`: 2 findings — `reliability/High` (bare except) and `maintainability/Medium` (unclosed file handle / resource leak) — a category the heuristic analyzer never checks for at all. Risk score 35, HIGH, no autofix.
- Gemini's fixes were both real and minimal: `mixed_issues.py` got `except:` → `except Exception:` with everything else untouched; `flaky_try_except.py` got wrapped in `with open(path) as f:` plus `except:` → `except Exception:` — a correct resource-leak fix, not something the heuristic fixer knows how to do.
- Risk reports: score 0-100, level low/medium/high, `should_autofix` bool, and a `reasons` list explaining every deduction (e.g. "High severity issue detected," "Return statements may have been removed").

---

## 4) Reliability and safety rules

Two rules from `assess_risk`:

1. **`"return" in original_code and "return" not in fixed_code"` → -30 points.**
   Checks whether the fix deleted every `return` statement. Matters because a fix that silently drops a return breaks the function's contract (callers get `None` instead of the ex­pected value) — a classic over-editing failure mode. False positive: a fix could legitimately replace `return` with `yield` or raise an exception instead, which is a valid rewrite but still gets penalized. False negative: it only checks for the literal substring `"return"` anywhere in the file — if the fixed code still contains a `return` somewhere else in an unrelated function, the check passes even though the specific function that needed fixing lost its return.

2. **`len(fixed_lines) < len(original_lines) * 0.5` → -20 points.**
   Flags fixes that shrink the code by more than half, a proxy for "the model deleted logic instead of fixing it." Matters because LLM fixers occasionally "solve" a bug by stubbing out the function. False positive: a legitimately verbose function (e.g. one with a lot of dead/commented code) could be justifiably shortened by 50%+ during a real cleanup and get flagged as risky. False negative: a fix that *adds* an equal number of lines of new bugs, or that keeps line count the same while gutting the logic inside (e.g. replacing a real branch with `pass`), passes this check untouched.

---

## 5) Observed failure modes

1. **Confirmed bug — "Heuristic only" mode wasn't actually heuristic for fixes.** The sidebar wired `MockClient()` into the agent for "Heuristic only (no API)" mode. `MockClient.complete()` returns non-JSON for the analyzer (correctly forcing heuristic fallback for issue detection), but returns a non-empty placeholder string (`"# MockClient: no rewrite available in offline mode."`) for the fixer step. Since `_can_call_llm()` only checks "is there a client with a `.complete` method," the agent treated this placeholder as a real LLM fix and never fell back to `_heuristic_fix`. Result on `print_spam.py`: risk assessor compared the original function against a one-line placeholder and correctly flagged it as a massive, return-stripping rewrite (score 45, MEDIUM, no autofix) — the *risk assessor* worked exactly right, but the *agent* was fixing nothing while claiming to be in a purely offline, deterministic mode. Fixed by changing `bughound_app.py` to pass `client=None` for that mode, which makes `_can_call_llm()` False and routes both analyze and fix through the real heuristic paths. This is a good example of a guardrail (risk_assessor) correctly catching a bug it wasn't even designed to catch, one layer up in the agent wiring.
2. **Bare except message understates the case:** heuristic analyzer's fixed regex text substitution for bare `except:` inserts a placeholder comment (`# [BugHound] log or handle the error`) rather than real handling — technically "fixed" per the risk scorer (bare except no longer literally present), but functionally still just swallows the exception silently. This is a case where the automated fix "feels complete" but isn't actually more reliable than the original bug.

3. **Gemini's issue-type taxonomy isn't fixed across runs.** Ran `flaky_try_except.py` through Gemini analysis 3 separate times and got 3 different category strings for the exact same bare-`except:` bug: `"reliability"` (lowercase), `"Error Handling"`, and `"Bare Except"`. This isn't just a casing quirk (the heuristic analyzer always returns a fixed Title Case vocabulary like `"Reliability"`, `"Code Quality"`) — Gemini is inventing a new category label essentially every call, with no fixed set to draw from. `_normalize_issues` in `bughound_agent.py` coerces every field to `str()` but never canonicalizes or validates against a known category list, so any downstream code doing exact-string comparisons or grouping by `type` (UI badges, category filters, category-specific risk rules) would treat these three runs as three unrelated issue kinds. Not currently exploited by `risk_assessor.py` (it only reads `severity`, not `type`), but it's a live landmine if that changes, and it already makes the "Detected issues" panel inconsistent for a human reader across repeated runs of identical code.

4. **Heuristic fixer silently drops data when rewriting multi-arg `print()` calls.** On `print_spam.py`, `_heuristic_fix`'s naive `print(` → `logging.info(` substitution turned `print("Hello", name)` into `logging.info("Hello", name)`. `logging.info` treats its second positional argument as a `%`-style format arg, not a second value to display — since `"Hello"` has no `%s` placeholder, `name` is silently swallowed and never appears in the log output. The risk assessor scored this fix as safe (95, LOW, autofix YES) because the diff is minimal and `return` is preserved — it has no way to catch a semantic change hiding inside an otherwise-clean-looking line. Confirmed this is conditional, not universal: the same fixer applied cleanly to `mixed_issues.py`'s single-arg `print("computing ratio...")`, with no data loss, because there was only one argument to begin with. This is the clearest example in testing of a fix that "looks safe" by every structural signal BugHound checks, while still being wrong.

5. **Gemini's severity/category labeling is not stable across runs on the identical bug.** Ran `flaky_try_except.py` through Gemini analysis twice. First run: the bare `except:` was labeled `reliability / High`, contributing a −40 point penalty (score 35, HIGH, no autofix). Second run, same file, same code: the identical bare `except:` was labeled `Error Handling / Medium`, contributing only −20 (score 55, MEDIUM, no autofix). The proposed fix was identical and correct in both runs (`with open(path) as f:` + `except Exception:`) — only the analyzer's severity judgment drifted, at temperature 0.2. Because `risk_assessor.py`'s score is a direct linear function of the severity string the analyzer returns, this means the *same bug* can swing between HIGH and MEDIUM risk purely from LLM non-determinism, not from any real difference in the code or the fix. This is the most consequential failure mode found: it undermines the assumption, baked into the risk-scoring formula, that a severity label is a stable, trustworthy signal rather than a probabilistic one.

---

## 6) Heuristic vs Gemini comparison

- Heuristics only catch the 3 patterns hard-coded into `_heuristic_analyze` (print, bare except, TODO) — perfectly consistent, zero false positives on unrelated code, but blind to anything not literally matching those substrings/regex (e.g. mutable default arguments, off-by-one loops, unclosed resources beyond the specific try/except shape).
- Confirmed live: Gemini caught the same bare-except and TODO issues heuristics did, but also flagged the unclosed `open()` file handle on `flaky_try_except.py` as a `maintainability/Medium` resource-leak risk — something no heuristic rule checks for at all. Its explanations were also more specific ("catches SystemExit and KeyboardInterrupt" vs. the heuristic's generic "catch a specific exception").
- Issue-type casing was inconsistent between the two sources (Title Case from heuristics, lowercase from Gemini in one run) — see failure mode #3 above. Both runs returned clean, parseable JSON with no wrapped prose in this testing, so the fallback-on-bad-JSON path wasn't exercised live (it was previously verified with `MockClient` in `tests/test_agent_workflow.py`).
- Heuristic fixes are narrow, minimal, and 100% predictable regex swaps, but limited to the 2 issue types they know how to fix (print, bare except) — a TODO issue, for instance, produces no code change at all in heuristic mode. Gemini's fixes on both test files were surprisingly minimal and correct: `except:` → `except Exception:` and wrapping the file read in `with open(path) as f:`, with no unrelated reformatting in either case.
- The risk scorer agreed with intuition on both live runs: score 45/MEDIUM (mixed_issues.py, 2 Low + 1 High issue) and score 35/HIGH (flaky_try_except.py, 1 High + 1 Medium) both correctly withheld autofix, even though both fixes were arguably fine to merge after a quick human read. This is the assessor being appropriately conservative — it downgrades any fix that touches a bare `except`, regardless of how clean the replacement is, which is the right default for a High-severity reliability issue.

---

## 7) Human-in-the-loop decision

Scenario: the agent detects a bare `except:` (High severity) *and* the proposed fix removes an existing `return` statement in the same diff — e.g. a fix that "handles" the exception by swallowing it and falling through without returning anything. This combination (high-severity issue + structural function-contract change) should always defer to a human, because a wrong guess here changes what every caller of that function receives on error, silently.

- Trigger: severity == High AND a `return` was removed in the diff.
- Implementation: `risk_assessor.py`, inside `assess_risk` — both signals already exist as independent penalties; the guardrail should combine them into a hard block on `should_autofix`, not just a score deduction.
- Message to show: "This fix touches error-handling logic and changes what the function returns. Auto-fix is disabled — please review this change manually before merging."

---

## 8) Improvement idea

Tighten the auto-fix threshold. Previously `should_autofix` was `True` whenever `score >= 75`; that let single-Medium-severity fixes (score 80) through automatically even though they came from an LLM rewrite. Changed the "low risk" cutoff from `score >= 75` to `score >= 90` in `risk_assessor.py`, and added a standalone guardrail so any High-severity issue blocks auto-fix outright regardless of computed score. Both changes are covered by new tests in `tests/test_risk_assessor.py` (`test_tightened_low_threshold_blocks_autofix_at_score_80`, `test_high_severity_issue_blocks_autofix_regardless_of_score`). This is low-complexity (two small conditionals) but measurably shifts the agent from "auto-fix by default unless clearly bad" to "auto-fix only when clearly safe" — the correct default for a system whose fixer can silently fall back to an untrusted LLM.
