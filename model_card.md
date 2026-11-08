# BugHound Mini Model Card (Reflection)

---

## 1) What is this system?

**Name:** BugHound
**Purpose:** Analyze a Python snippet, propose a fix, and run reliability checks before suggesting whether the fix should be auto-applied.

**Intended users:** Students learning agentic workflows and AI reliability concepts. It is a teaching tool, not a production code reviewer: its suggestions should be read, not trusted blindly.

---

## 2) How does it work?

BugHound runs a five-step loop in `BugHoundAgent.run()`:

1. **Plan:** logs that it will do a scan + fix proposal. (This step is fixed; it does not adapt to the input.)
2. **Analyze:** finds issues as a list of `{type, severity, msg}`.
   - **Gemini:** sends `prompts/analyzer_*.txt` and expects a bare JSON array.
   - **Heuristics:** used when there is no client, the API call fails, or the model output can't be used. They check for real `print()` calls (parsed with `ast`), bare `except:`, and `TODO`.
   - Model output is checked before it is accepted: each issue needs a non-empty `type` and `msg` and a severity of Low/Medium/High. Invalid items are rejected and logged. If the model returned issues but none were valid, BugHound falls back to heuristics. An empty `[]` is accepted as "no issues".
3. **Act:** proposes a fix. With no issues, the code is returned unchanged. Otherwise Gemini rewrites the code using `prompts/fixer_*.txt` ("preserve behavior", "smallest changes"), or the heuristic fixer does text replacements (`print(` → `logging.info(`, `except:` → `except Exception as e:`).
4. **Test:** `reliability/risk_assessor.py` scores the fix from 0 to 100 and assigns a level: low (75+), medium (40–74) or high (below 40).
5. **Reflect:** auto-apply is recommended only if the risk level is low **and** no behavior-change signal fired. Otherwise the tool asks for human review.

**Heuristic vs Gemini in practice:** the Streamlit "Heuristic only" mode passes a `MockClient`, not `None`. The mock returns non-JSON for analysis, so analysis falls back to heuristics. The fixer step, however, still calls the mock (see Failure mode 3).

---

## 3) Inputs and outputs

**Inputs:** short single-function scripts:

| Input | Shape |
|---|---|
| `cleanish.py` | function that already uses `logging` |
| `print_spam.py` | function with 3 `print()` calls behind a `verbose` flag |
| `mixed_issues.py` | `TODO` comment + `print()` + bare `except:` around a division |
| `flaky_try_except.py` | file read inside a bare `except:` |
| Edge cases | empty file; comments only (`# TODO`, `# never use print( in prod`); `print(` inside a string literal |

**Outputs observed:**

| Input | Mode | Issues | Fix | Risk report |
|---|---|---|---|---|
| `cleanish.py` | Heuristic | none | unchanged | 100 / low / auto-fix |
| `mixed_issues.py` | Gemini | TODO (Low), print (Low), bare except (High) | added `validate_input()` that raises `TypeError`, `logger.info`, `except ZeroDivisionError` | 15 / high / review |
| `print_spam.py` | Gemini | print (Low) | `print` → `logger.info` with a module logger | 80 / low / **review** (was 95 / auto-fix before the behavior-change rule) |
| `mixed_issues.py` | Heuristic (client=None) | print (Low), bare except (High), TODO (Medium) | `import logging`, `logging.info`, `except Exception as e:` | high / review |
| empty file | Heuristic | none | unchanged | 0 / high / "No fix was produced." |
| `print(` in a string | Heuristic | none (after the guardrail) | unchanged | 100 / low |

---

## 4) Reliability and safety rules

**Rule A: Severity deductions (High −40, Medium −20, Low −5)**
- *Checks:* how serious the detected issues are.
- *Why it matters:* serious issues tend to need bigger, riskier fixes.
- *False positive:* a High-severity bare `except:` with a trivially correct one-line fix still gets blocked.
- *False negative:* it scores the **problem**, not the **fix**. Gemini's `print_spam` fix had only one Low issue, so it scored 95 and was auto-applied, even though it silently removed the user-visible greeting. Severity also comes from the model, which can mislabel it.

**Rule B: "Return statements may have been removed" (−30)**
- *Checks:* `"return"` appears in the original but nowhere in the fix.
- *Why it matters:* deleting returns changes what callers receive.
- *False positive:* a comment or string containing the word "return" counts as a return statement.
- *False negative:* changing **what** is returned is invisible. `return 0` → `return None`, or Gemini's change that makes `compute_ratio("a", 1)` raise instead of returning `0`, both pass this check because some `return` still exists.

**Rule C (added): Observable behavior change (−15 each, blocks auto-fix)**
- *Checks:* the fix has fewer `print(` occurrences or more `raise` statements than the original.
- *Why it matters:* these change what users see or what callers have to handle, so they always go to a human.
- *False positive:* the comments-only file is flagged as "removes print() output" because the original mentions `print(` in a comment.
- *False negative:* behavior changes made any other way (changed return values, narrowed `except` types, `sys.exit`) are not detected.

---

## 5) Observed failure modes

1. **False positive that changed behavior (fixed with a guardrail).**
   `def help_text(): return "Call print(x) to debug."`. The text-based heuristic flagged a "print statement", and the heuristic fixer rewrote the string to `"Call logging.info(x) to debug."`, changing the function's return value. The same thing happened to the comment `# never use print( in prod`.

2. **Over-editing by Gemini.**
   For `mixed_issues.py`, asked to make "the smallest changes" and "preserve behavior", Gemini added a new `validate_input()` function that raises `TypeError` and narrowed `except:` to `except ZeroDivisionError`. `compute_ratio("a", 1)` used to return `0` and now crashes. For `print_spam.py`, switching to `logger.info` means the greeting no longer prints by default. The original scorer auto-applied that fix (95 / low).

3. **Heuristic mode in the app replaces the whole file.**
   With `MockClient`, analysis correctly falls back to heuristics, but the fixer still calls the mock. The mock returns `# MockClient: no rewrite available in offline mode.`, so the proposed "fix" is a one-line comment. The risk scorer catches it (`print_spam.py` → 30 / high, "much shorter", "returns removed"), but the diff shown to the user deletes all of their code.

4. **Misleading report on empty input.** An empty file is scored 0 / high with "No fix was produced." This is safe, but the reason is wrong: there was nothing to fix.

5. **Format failures are invisible in Gemini mode.** `GeminiClient.complete()` catches API errors and returns `""`. The agent then logs "LLM output was not parseable JSON", so a rate limit looks like a format failure.

---

## 6) Heuristic vs Gemini comparison

- **Gemini detected more context:** it explained *why* the bare except is dangerous (it swallows `KeyboardInterrupt`/`SystemExit`) and connected the TODO to missing input validation. Heuristics only know three patterns.
- **Heuristics were consistent:** the same three issue types every time, with fixed severities (print = Low, TODO = Medium, bare except = High). Gemini rated the TODO as Low and used lowercase type labels (`"readability"`).
- **Fixes differed a lot:** heuristic fixes are mechanical text swaps (which caused failure mode 1). Gemini's fixes read like real refactors (module logger, `%s` formatting, a specific exception type) but went past the requested minimal change and altered behavior.
- **Risk scorer vs intuition:** originally it disagreed with my intuition on `print_spam.py`, auto-applying a fix that hides output. After adding Rule C it matches: both Gemini fixes go to human review.

---

## 7) Human-in-the-loop decision

**Scenario:** a fix changes exception-handling behavior, for example narrowing `except:` to `except ZeroDivisionError` or adding a new `raise`. Inputs the old code handled quietly would now crash the caller.

- **Trigger:** the set of exception types caught, or the number of `raise` statements, differs between the original and the fix. Compare `ast.ExceptHandler` and `ast.Raise` nodes rather than text. (The `raise` part is already implemented in Rule C.)
- **Where:** in `risk_assessor.py`. It is the single gate every fix passes through, heuristic or Gemini, and it already owns `should_autofix`. The UI should only display the decision.
- **Message:** "⚠️ This fix changes which errors are caught or raised. Code that used to return a fallback value may now raise an exception. Please review the error handling before applying."

---

## 8) Improvement idea

**Make the fixer fall back to heuristics whenever analysis did.** Today, if analysis falls back (non-JSON output, API error), the fixer still asks the same unreliable client and gets back junk (failure mode 3). Add a flag in `analyze()` (e.g. `self._used_fallback = True`) and have `propose_fix()` use `_heuristic_fix` when it is set.

- **Complexity:** about 5 lines in `bughound_agent.py`, no new dependencies.
- **Measurable:** add an offline test with `MockClient` on `print_spam.py` asserting `fixed_code` still contains `def greet` and `return True`. It fails today (the fix is a one-line comment) and passes with the change. In the app, heuristic mode would show a real diff instead of deleting the file.
