# BugHound Mini Model Card (Reflection)

---

## 1) What is this system?

**Name:** BugHound
**Purpose:** Analyze a Python code snippet for reliability, maintainability, and code-quality issues; propose a targeted fix; evaluate the risk of that fix; and decide whether to apply it automatically or defer to a human reviewer.

**Intended users:** Students learning agentic AI workflows and software engineering teams who want a lightweight first-pass review tool before a human code review. BugHound is deliberately scoped to Python snippets — it is not a full static analysis suite.

---

## 2) How does it work?

BugHound runs a five-step agentic loop on every submission:

**PLAN** — The agent decides its overall strategy (scan for issues, propose a minimal fix, then evaluate safety) and logs the intent. No external calls happen here.

**ANALYZE** — Issue detection. In *Heuristic mode*, three regex rules are applied: bare `except:` (High severity), `print(` as a standalone call (Low severity), and `# TODO` comments (Medium severity). In *Gemini mode*, the agent sends the code to the Gemini API with a structured prompt requesting a JSON array of issue objects. If the API fails, returns malformed JSON, or returns issues with non-standard severity values, the agent automatically falls back to heuristics and logs the reason.

**ACT** — Fix generation. Heuristic mode uses deterministic pattern replacements (converts bare `except:` to `except Exception as e:`, replaces standalone `print(` calls with `logging.info(`, prepends `import logging`). Gemini mode asks the model to rewrite the full file with minimal changes. If the model returns empty output or a non-code response, the agent falls back to heuristics.

**TEST** — The proposed fix is passed to `reliability/risk_assessor.py`, which scores it from 0 to 100 by deducting points for detected issue severity, structural changes (code length reduction, missing return statements, new import introductions), and modifications to exception-handling patterns.

**REFLECT** — If the risk score is ≥ 75 (level "low"), `should_autofix` is set to `True` and the UI offers to apply the fix. Otherwise, the agent flags the fix for human review. The full decision trace is written to the Agent Trace log visible in the UI.

---

## 3) Inputs and outputs

**Inputs tested:**

| File | Shape | Issues present |
|---|---|---|
| `cleanish.py` | 5-line function using `logging` correctly | None |
| `print_spam.py` | 8-line function with 3 `print()` calls | 1 Low (print statements) |
| `flaky_try_except.py` | 10-line function with a bare `except:` block | 1 High (bare except) |
| `mixed_issues.py` | 9-line function with TODO + print + bare except | 3 issues (Low, Medium, High) |
| `print(` inside string | 3-line function returning a string containing `print(` | 0 (after guardrail fix) |

**Outputs observed:**

- *Issues detected:* Objects with `type` (e.g., "Reliability"), `severity` ("Low"/"Medium"/"High"), and `msg` (plain-language explanation). In heuristic mode, at most one issue per pattern. In Gemini mode, the model may return several finer-grained issues for the same code.
- *Fixes proposed:* In heuristic mode, the fix is a direct line-for-line rewrite (new import header, replaced calls, expanded except clause). In Gemini mode, the model may reorganize the function, add docstrings, or introduce type hints — changes beyond the stated issues.
- *Risk reports:* Range from score 100 / level "low" / autofix=True (cleanish.py, no issues) to score 0–30 / level "high" / autofix=False (mixed_issues.py, where High + Medium severities and a new import signal drove the score to 5).

---

## 4) Reliability and safety rules

**Rule 1 — Return statement removal penalty (−30 points)**

*What it checks:* Whether `return` appears in the original code but is absent from the fixed code entirely.

*Why it matters:* A fix that removes all return statements from a function changes its return value from the intended type to `None`. This is a silent behavioral regression that no syntax checker would catch.

*False positive:* Code that uses `return` only in a comment or docstring (e.g., `"""Returns a string"""`) would still satisfy `"return" in original_code`, so removing the docstring in the fix could trigger the penalty even if no real return was removed.

*False negative:* If the original has two `return` statements and the fix removes one of them (but keeps the other), the string `"return"` still appears in both versions. The check never fires. The function's control flow may have changed significantly and the agent reports no concern.

**Rule 2 — New import introduction penalty (−25 points, added during this activity)**

*What it checks:* Whether the fixed code contains `import` or `from` lines that were not present in the original.

*Why it matters:* A new import is a structural dependency change. It can fail at runtime (`ImportError`), introduce unexpected module-level side effects, or create a name collision. The heuristic fixer always adds `import logging` when replacing print statements — a change the original risk assessor did not scrutinize at all.

*False positive:* Adding `import logging` is almost always safe. The penalty flags it as requiring human review even when the import is benign, which may feel overly cautious for a simple print-to-logging conversion.

*False negative:* An import that is moved (present in both versions but on a different line, e.g., moved from mid-file to the top) would not be detected since the set difference would be empty.

---

## 5) Observed failure modes

**Failure mode 1 — False positive and string corruption from substring print detection**

*Snippet:*
```python
def explain():
    return "Use print(x) to display output"
```

*What went wrong:* The original heuristic used `if "print(" in code` — a raw substring scan with no awareness of string context. The string `"Use print(x) to display output"` contains the substring `print(` but it is not a function call. The agent flagged a Code Quality issue and the heuristic fixer then applied `str.replace("print(", "logging.info(")` across the entire file, corrupting the string to `"Use logging.info(x) to display output"`. The fixed code would return a different string than the original — a silent behavioral regression introduced by the agent itself.

*Resolution:* Detection and replacement were both changed to use a line-anchored regex (`^\s*print\s*\(` with `re.MULTILINE`), so only `print(` appearing as the start of a statement line is matched. A regression test was added to prevent recurrence.

**Failure mode 2 — LLM severity values silently bypassing risk scoring**

*What went wrong:* The `_normalize_issues` method accepted any string as a severity value. When Gemini returns `"severity": "Critical"` or `"severity": "warning"` (both observed in practice), these strings pass through normalization unchanged. The risk assessor then checks `severity.lower() == "high"` — which does not match `"critical"` — so the issue receives zero risk penalty. A "Critical" severity issue gets the same score deduction as having no issues at all, and the fix could receive an auto-apply approval it should not have received.

*Resolution:* An `_issues_are_valid()` check was added to the parser. Before accepting any LLM output, every issue's severity must be exactly `"Low"`, `"Medium"`, or `"High"`. Any other value causes the parser to return `None`, triggering a logged fallback to heuristics.

---

## 6) Heuristic vs Gemini comparison

**What Gemini detected that heuristics did not:**
- Missing input validation (e.g., no check for `y == 0` before division in `compute_ratio`)
- Implicit type assumptions (parameters used as numbers with no type annotation or guard)
- Missing `f.close()` / context manager issues even when no bare except was present
- Stylistic issues with variable naming that heuristics have no rules for

**What heuristics caught consistently:**
- Every bare `except:` clause reliably flagged as High severity
- Every `print(` function call (after the guardrail fix) reliably flagged as Low severity
- `# TODO` comments always detected as Medium severity
- Results were 100% reproducible — same input always produced the same output

**How proposed fixes differed:**
- Heuristic fixes are minimal and mechanical: add one import header, replace one pattern. The diff is always small and easy to audit.
- Gemini fixes tended to go further: add type hints, rewrite the try/except structure, add a docstring, or restructure the function. The diff was larger and sometimes changed more than the detected issues warranted.

**Did the risk scorer agree with intuition?**
- For heuristic fixes: mostly yes. A print-only fix scoring 95 before the import guardrail felt too permissive — adding `import logging` is structural, not trivial.
- For Gemini fixes with over-editing: the risk scorer partially caught it via the length-ratio check, but only if the rewrite shrank the code significantly. A Gemini fix that *expanded* the code with new docstrings would not trigger any structural penalty under the original rules.

---

## 7) Human-in-the-loop decision

**Scenario:** The user submits `flaky_try_except.py`. Gemini detects the bare except, proposes a fix that replaces `except:` with `except Exception as e: logging.error(e)`, and also refactors the file-reading logic to use a `with` statement instead of explicit `open`/`close` calls. The fix is syntactically valid and the risk score is 50 (medium).

**Why the agent should not auto-fix:** The fix changed two things — the exception handler and the resource management pattern. Even if both changes are improvements, combining them means a reviewer must evaluate two independent behavioral changes at once. The `with` statement change is not related to any detected issue; the agent acted beyond its mandate.

**Trigger to add:** In `assess_risk`, count the number of semantically distinct change types in the diff (e.g., new control flow keywords, new method calls on file handles, new import statements). If more than one structural change pattern is present alongside a High or Medium severity issue, increase the risk score deduction and add a reason: `"Fix appears to address more than the detected issues. Review scope before applying."` Set `should_autofix = False` regardless of score.

**Where to implement:** Primarily in `risk_assessor.py` so it applies to both heuristic and Gemini fixes. The UI should then display the specific reason text so the user understands *why* human review is required — not just that it is.

**Message to show the user:**
> "BugHound detected changes beyond the reported issues. Please review the diff before applying — the fix may be correct, but it changes more than was flagged."

---

## 8) Improvement idea

**Load prompt files from `prompts/` at runtime instead of using hardcoded inline strings.**

Currently, `bughound_agent.py` embeds the analyzer and fixer prompts directly as string literals (lines 62–69 and 97–105). The `prompts/` folder contains more detailed, well-structured versions of the same prompts — including explicit output format requirements, a list of issue patterns to look for, and guidance on handling ambiguous cases — but these files are never loaded. Iterating on prompt quality requires editing Python source code, which makes it harder to test prompt changes in isolation and risks introducing Python syntax errors when adjusting natural language text.

**The change:** Replace the inline string literals with `Path("prompts/analyzer_system.txt").read_text()` and equivalent calls at agent initialization (with a fallback to the current inline strings if the files are missing). This decouples prompt engineering from code changes, allows prompt changes to be reviewed in plain text diffs, and makes the richer constraints in the existing prompt files take effect immediately — particularly the explicit enumeration of pattern types and the stricter "no commentary" instructions that the current inline prompts omit.

**Why it is low complexity:** It is a file read, a fallback, and a variable substitution. It requires no new dependencies, no new agent logic, and no changes to the risk assessor or tests. The change is fully reversible by removing the file reads and restoring the inline strings.
