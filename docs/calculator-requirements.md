# Calculator requirements — V1 simple, V2 scientific

## Scope and chosen defaults

Build one browser calculator in two increments. For a new repository, use React + TypeScript + Vite, Vitest, and React Testing Library; reuse existing repository conventions if present. The calculation engine is independent of React/DOM and testable as pure functions. No backend, login, database, or cloud deployment.

This document is the source of truth for backlog creation. Story IDs below are **requirement identifiers, not Azure DevOps item IDs**. Actual numeric work-item IDs and URLs must come from the server and be saved to [work-item-map.json](work-item-map.json). Priority P1 means first within its version; P2 means next. Do not infer numeric priority fields or estimates from these labels.

Stable demo key: `copilot-calculator-demo`.

## Compact backlog

| Version | Requirement ID | Kind | Suggested title | Priority | Dependencies |
|---|---|---|---|---|---|
| V1 | CALC-V1 | Version parent | V1 — Simple calculator | P1 | None |
| V1 | CALC-V1-ENGINE | Story | Build safe arithmetic expression engine | P1 | None |
| V1 | CALC-V1-UI | Story | Build accessible calculator input and display | P1 | CALC-V1-ENGINE |
| V1 | CALC-V1-QUALITY | Story | Add keyboard, responsive behavior, and V1 validation | P2 | CALC-V1-ENGINE, CALC-V1-UI |
| V2 | CALC-V2 | Version parent | V2 — Scientific extension | P1 | All V1 stories |
| V2 | CALC-V2-ALGEBRA | Story | Extend parser with grouping, powers, and roots | P1 | All V1 stories |
| V2 | CALC-V2-TRIG | Story | Add scientific mode and DEG/RAD trigonometry | P1 | CALC-V2-ALGEBRA |
| V2 | CALC-V2-FUNCTIONS | Story | Add logs, exponential, factorial, and constants | P2 | CALC-V2-ALGEBRA, CALC-V2-TRIG |

Default Agile mapping is Feature → User Story. Use Product Backlog Item, Requirement, or Issue only after verifying the target process. On Basic, use a supported Issue parent or standalone Issues rather than inventing a Feature. Keep the requirement IDs and version even if the hierarchy falls back. Tasks under stories are optional and should not duplicate the story's entire scope.

A version parent summarizes its child scope; it is not another item for implementing the same work. Parent acceptance is all of its story acceptance criteria plus the version exit checks below. Model cross-version dependencies with a supported predecessor/successor relation when available; otherwise preserve them in description and map. Do not mistake related links for enforced execution order.

## Shared calculation contract

- **Grammar:** explicit numeric literals, supported operators, and the functions introduced in each version. Numbers accept digits and at most one decimal point: `12`, `12.5`, `.5`, `5.`. Unary `+`/`-` allow signed numbers. Reject bare `.`, duplicated decimals, adjacent operands, unknown identifiers, and incomplete expressions on evaluation.
- **V1 precedence:** unary signs, then `*` and `/`, then `+` and `-`. Binary operations of equal precedence associate left-to-right. Thus `2+3*4 = 14`, `8/4*2 = 4`, `-2*3 = -6`. No parentheses or scientific functions in V1.
- **Equals:** evaluate the complete visible expression, not a hidden immediate-execution accumulator. After a successful equals, a digit/decimal starts a new expression; a binary operator continues from the internal, unrounded result. Repeated equals leaves the same result; it does not repeat the last operation. Backspace after equals returns to editing the prior expression, removes its last character, and clears the result state. Sign change after equals negates the result for continued calculation.
- **Editing:** keypad and editable expression field stay synchronized. Backspace removes the character before the caret (or the selected text). Sign change in an editable expression toggles the leading unary minus of the numeric literal at the caret; when there is no number there, start a negative numeric literal at that position. In V2 it does not silently negate a whole compound expression: use `-(...)` explicitly. Clear resets expression, result, and error but preserves V2 mode/angle settings.
- **Evaluation safety:** implement tokenization and parsing, not JavaScript evaluation. Never use `eval`, `new Function`, or code execution based on input. Expressions are data, not program instructions.
- **Bounds:** max 256 input characters, max 128 tokens, max parenthesis depth 16 in V2. Reject an edit that exceeds an input bound with a readable message and preserve the existing input. Return a recoverable error if parsing/evaluation exceeds a bound.
- **Precision:** use full finite IEEE-754 double values internally. Format the display to at most 12 significant digits, trim unnecessary trailing fractional zeros, and show actual negative zero as `0`. Use scientific notation for nonzero magnitudes below `1e-6` or at least `1e12`. Never round intermediate results merely to match the display. Display-generated exponent notation need not be accepted as editable input; operator-continuation from a result uses its internal value, not reparsing its formatted text.
- **Comparisons:** engine assertions pass if `abs(actual - expected) <= max(1e-10, 1e-12 * abs(expected))`. Use exact checks for error classifications, settings, and UI strings whose formatting is specified. Do not use loose equality or exact binary floating-point equality for ordinary decimal results.
- **Error handling:** distinct user-facing errors for invalid/incomplete input, division by zero, mathematical domain violations, and overflow/non-finite results. Do not display `NaN`, `Infinity`, stack traces, or stale results as success. Keep the expression available for correction. A valid edit and subsequent equals can recover; clear always recovers. No prior result is reused silently after a failed evaluation.
- **Safety of simple scope:** percent, memory, history, persistence, unit conversion, implicit multiplication (`2pi`, `2(3+4)`), complex numbers, arbitrary variables, and repeated-equals arithmetic are not in scope. Scientific input uses explicit multiplication, named functions, and the grammar below. Do not add them as surprise features.

## V1 — simple calculator

### CALC-V1-ENGINE — safe arithmetic engine

**User story:** As a calculator user, I want arithmetic expressions evaluated predictably so I can trust the result and recover from invalid input.

**Requirements:** V1-R01 numeric/decimal/signed input; V1-R02 four operations and precedence; V1-R03 safe errors, bounded parsing, and readable precision; V1-R04 engine isolated from UI.

**Acceptance criteria:**

- **V1-E1 (V1-R01):** Given valid numeric literals and unary signs, when the engine evaluates `.5+1.25`, `5.+.5`, and `-2*3`, then the results are `1.75`, `5.5`, and `-6` within the shared tolerance. Given `1..2`, bare `.`, or `2+`, when evaluated, then it returns an invalid/incomplete-input error rather than a numeric result.
- **V1-E2 (V1-R02):** Given arithmetic expressions, when evaluating `2+3*4`, `8/4*2`, `10-3-2`, and `8/-2`, then results are `14`, `4`, `5`, and `-4`. Given `2+3*4`, when equals is requested, then the whole expression is evaluated with precedence, not left-to-right immediate execution.
- **V1-E3 (V1-R03):** Given `9/0` or `0/0`, when evaluated, then a division-by-zero error is returned. Given a subsequent valid `6/2`, then the result is `3`; no stale prior result is returned. Given unsupported `(2+3)`, `sin(30)`, `2pi`, or executable-looking text, when evaluated in V1, then it is rejected.
- **V1-E4 (V1-R03):** Given `0.1+0.2`, when evaluated and formatted, then the display is `0.3`. Given `1/3`, then the display is `0.333333333333`. Given an overflowing multiplication or a bound violation, then a recoverable error is returned, never a non-finite success.
- **V1-E5 (V1-R04):** Given the engine tests, when run without mounting React or a browser DOM, then parsing, evaluation, error classifications, and formatting can be tested. Given an input that would be JavaScript source, then the engine treats it as invalid data and does not execute it.

**Implementation boundaries:** Engine and format tests only; a minimal app scaffold may be needed in a fresh repository. Do not build all the UI or scientific functions under this story.

### CALC-V1-UI — accessible input and display

**User story:** As a user, I want clear input, output, and editing controls so I can enter and correct calculations.

**Requirements:** V1-R05 digit/operator/decimal keypad and editable field; V1-R06 equals/clear/backspace/sign change; V1-R07 labels, focus, and recoverable display errors.

**Acceptance criteria:**

- **V1-U1 (V1-R05):** Given the calculator, when entering `12.5+7.5` through the keypad or expression field and pressing equals, then the input remains understandable and the displayed result is `20`. Field and keypad operate on the same expression and do not diverge.
- **V1-U2 (V1-R06):** Given input `123`, when backspace is activated at the end, then input becomes `12`. Given `12`, when sign change is activated on that literal twice, then input changes to `-12` and back to `12`. Given a selected portion of input, backspace removes only that selection. Clear empties expression/result/error.
- **V1-U3 (V1-R06):** Given `2+3` evaluated to `5`, when equals is pressed again, then the result stays `5`. When a digit `7` is then entered, a new expression `7` begins. When `*2` is entered after a fresh result `5`, equals yields `10` using the internal result.
- **V1-U4 (V1-R07):** Given an error from `9/0`, when the user edits it to `9/3` and evaluates, then the error clears and the display shows `3`. Invalid evaluation must not masquerade as a successful old result. A text error is announced using an appropriate live region without stealing focus.
- **V1-U5 (V1-R07):** Given buttons and inputs, when inspected via their accessibility roles, then each has an accessible name, native keyboard activation, visible focus, and logical focus order. Operator-only visual symbols have understandable labels such as Multiply and Divide.

### CALC-V1-QUALITY — keyboard, responsiveness, and V1 validation

**User story:** As a keyboard or mobile user, I want the same calculator behavior without a mouse and without layout overflow.

**Requirements:** V1-R08 keyboard parity; V1-R09 responsive presentation; V1-R10 mapped tests and usage notes.

**Acceptance criteria:**

- **V1-Q1 (V1-R08):** Given calculator focus, when digits, `.`, `+`, `-`, `*`, and `/` are typed, then the same valid input is available as with the keypad. Enter or `=` evaluates, Escape clears, and Backspace edits. Outside the calculator, do not intercept shortcuts; do not suppress Tab navigation or browser shortcuts.
- **V1-Q2 (V1-R09):** Given viewports 320 px, 768 px, and 1280 px wide, when the calculator is rendered, then controls and error text are usable without page-level horizontal overflow. Long expressions may scroll inside their field. Interactive controls have target sizes at least 44×44 CSS px; text contrast is at least 4.5:1 and visible focus remains present.
- **V1-Q3 (V1-R10):** Given the V1 code, when the actual configured tests, type check, lint if configured, and build are run, then results are reported honestly and test names/map identify the V1 criteria they cover. A manual check documents keyboard and responsive behavior that automated tests do not establish.

**V1 exit:** All V1 stories' criteria are met, their validation is observed, and the README explains local start/test/build and the deliberate omissions. Report incomplete checks instead of marking the version done.

## V2 — scientific extension, not a rewrite

V2 extends the existing V1 engine and UI. Default mode on a fresh app load is **Simple**, and the scientific angle setting defaults to **DEG**. Toggling modes does not discard current input or results. Simple mode hides scientific controls and rejects scientific expressions on equals with guidance to switch modes; it does not erase them. Scientific mode supports all V1 operations. Angle choice persists during the current app session and across mode toggles, not across reloads (no persistence).

**V2 grammar and precedence (highest first):** explicit grouping/function calls and constants; postfix factorial; power `^` (right-associative); unary signs; multiplication/division; addition/subtraction. This means `2^3^2 = 512`, `-2^2 = -4`, `(-2)^2 = 4`, and `2^-2 = 0.25`. Exponent parsing must allow a signed exponent without making power left-associative. `square(x)` and `sqrt(x)` are named functions; square and root buttons insert these functions. Trig/log/exp functions require parentheses, for example `sin(30)` and `ln(e)`. Constants are `pi` and `e` and require explicit multiplication. Functions take one argument; no implicit multiplication or comma-separated arguments.

### CALC-V2-ALGEBRA — grouping, powers, and roots

**User story:** As a scientific-calculator user, I want grouping and algebraic operations without changing basic arithmetic behavior.

**Requirements:** V2-R01 parentheses/precedence; V2-R02 power/square/root; V2-R03 parser and numerical boundaries.

**Acceptance criteria:**

- **V2-A1 (V2-R01):** Given scientific mode, when `(2+3)*4` and `2+3*4` are evaluated, then results are `20` and `14`. Given mismatched or empty parentheses, then a recoverable invalid-input error is shown. Implicit multiplication `2(3+4)` is rejected.
- **V2-A2 (V2-R02):** Given scientific expressions, when evaluating `2^3`, `2^3^2`, `square(5)`, `sqrt(9)`, `-2^2`, `(-2)^2`, and `2^-2`, then results are `8`, `512`, `25`, `3`, `-4`, `4`, and `0.25` within tolerance.
- **V2-A3 (V2-R02/V2-R03):** Given `sqrt(-1)`, `0^0`, `0^-1`, or a negative base raised to a non-integer power, then a domain error is returned. `(-2)^3` succeeds with `-8`; negative bases are supported only with integer exponents. Given overflow such as `10^309`, then an overflow/non-finite error is returned.
- **V2-A4 (V2-R03):** Given more than 16 levels of nesting or the shared character/token bounds, then the calculator rejects it safely and remains responsive; subsequent valid input recovers. Existing V1 engine and UI tests remain in place and pass when actually run.

**Boundary:** Engine grammar and algebra UI controls can be added under this story. The general mode toggle is completed by the next story; test the extended engine directly until then. Do not claim the whole V2 UI is finished here.

### CALC-V2-TRIG — scientific mode and angles

**User story:** As a user, I want to choose degrees or radians and understand undefined trig results.

**Requirements:** V2-R04 Simple/Scientific toggle; V2-R05 DEG/RAD sin/cos/tan; V2-R06 accessible scientific controls and angle display.

**Acceptance criteria:**

- **V2-T1 (V2-R04/V2-R06):** Given a fresh app load, then Simple mode is selected and DEG is the scientific default. When switching to Scientific, then algebra/trig controls and a clearly labeled angle selector are available. Switching modes preserves the expression/result and current angle selection. Evaluating scientific-only input in Simple mode shows switch-mode guidance, without erasing input.
- **V2-T2 (V2-R05):** Given DEG, when evaluating `sin(30)`, `cos(60)`, and `tan(45)`, then results are `0.5`, `0.5`, and `1`. Given RAD, when evaluating `sin(pi/2)` and `cos(0)`, then results are `1` and `1`. Support the `pi` constant here because it is required for these tests; the final story completes constant controls and `e`.
- **V2-T3 (V2-R05):** Given the converted angle, when `abs(cos(angle)) < 1e-12`, then `tan` returns an undefined/domain error, not a huge misleading finite number. Thus DEG `tan(90)` and RAD `tan(pi/2)` error. `tan(89)` in DEG remains finite. For sin/cos only, magnitudes below `1e-12` may be normalized to exact zero to suppress representational residue; do not suppress small results for all functions.
- **V2-T4 (V2-R06):** Given a keyboard/screen-reader user, then mode, angle setting, and scientific buttons have accessible names, focus states, and keyboard operation. Controls do not introduce page-level horizontal overflow at the V1 viewport sizes. All V1 behavior remains available in Scientific mode.

### CALC-V2-FUNCTIONS — logs, exponential, factorial, and constants

**User story:** As a scientific-calculator user, I want common functions with clearly defined domains and trustworthy boundaries.

**Requirements:** V2-R07 log10/ln/exp; V2-R08 factorial; V2-R09 pi/e controls; V2-R10 complete V1/V2 regression coverage and documentation.

**Acceptance criteria:**

- **V2-F1 (V2-R07):** Given scientific mode, when evaluating `log10(100)`, `ln(e)`, and `exp(0)`, then results are `2`, `1`, and `1`. Logarithms require strictly positive real arguments; `log10(0)` and `ln(-1)` return domain errors. `exp(710)` returns an overflow error; finite underflow to zero is allowed and documented.
- **V2-F2 (V2-R08):** Given postfix factorial, when evaluating `0!` and `5!`, then results are `1` and `120`. Accept non-negative integers `0..170` only; `170!` is finite, `171!` returns a range error, and `(-1)!` or `2.5!` return domain errors. Due to precedence, use `(-1)!` to test negative input: `-1!` means `-(1!)`. Repeated factorial `5!!` is unsupported and rejected, not double-factorial notation.
- **V2-F3 (V2-R09):** Given constants entered using controls or explicit text, when evaluating `2*pi` and `e`, then full internal `Math.PI`/`Math.E` equivalent values are used. `pi`/`e` remain identifiable in input; `2pi` is rejected. `pi` displays `3.14159265359` using the shared 12-significant-digit rule.
- **V2-F4 (V2-R10):** Given the final V2 app, when the full V1 and V2 engine/UI suites, available type/lint checks, and build are actually run, then results are reported by command with pass/fail/not-run and criterion coverage. Document mode/angle defaults, grammar, domains, precision, and deliberate omissions. A failed or unrun regression prevents a verified-complete claim.

**V2 exit:** All V2 criteria and all V1 regressions are met. No rewrite, lost editing behavior, inaccessible scientific controls, silent domain coercion, or unsupported surprise features.

## Small presenter test deck

| Mode / setting | Input or action | Expected |
|---|---|---|
| V1 Simple | `2+3*4` | `14` |
| V1 Simple | `0.1+0.2` | display `0.3` |
| V1 Simple | `9/0`, then edit to `9/3` | error, then `3` |
| V1 Simple | `2+3`, equals twice | `5`, then unchanged `5` |
| V2 Scientific | `(2+3)*4` | `20` |
| V2 Scientific | `sqrt(9)` | `3` |
| V2 Scientific, DEG | `sin(30)` | `0.5` |
| V2 Scientific, RAD | `sin(pi/2)` | `1` |
| V2 Scientific, DEG | `tan(90)` | undefined/domain error |
| V2 Scientific | `log10(100)` / `ln(e)` | `2` / `1` |
| V2 Scientific | `5!` / `171!` | `120` / range error |

These are expected examples, **not a claim that a supplied app passed them**. Apply the shared engine tolerance and exact error/display checks. The story criteria, not just this short deck, define completion.

## References

- [GitHub repository agent skills](https://docs.github.com/en/copilot/how-tos/copilot-on-github/customize-copilot/customize-cloud-agent/add-skills)
- [Azure DevOps MCP setup and consolidation](https://github.com/microsoft/azure-devops-mcp)
- [Current local tools and actions](https://github.com/microsoft/azure-devops-mcp/blob/main/docs/TOOLSET.md)

Calculator behavior and backlog size above are chosen demo requirements, not claims derived from those platform references.
