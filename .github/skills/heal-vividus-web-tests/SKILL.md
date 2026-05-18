---
name: heal-vividus-web-tests
description: 'Automatically heal failing VIVIDUS web test stories by diagnosing failures, checking actual page state with Playwright, and fixing broken locators, changed text content, missing waits, or outdated steps. Use when: tests are failing after UI changes, locators are broken, page content has changed, elements were moved or renamed.'
argument-hint: 'Paste failing test output or specify which story to heal...'
---

## Process Overview

1. **Identify** failing test(s) and extract failure details
2. **Locate** the story file(s) containing the failing step(s)
3. **Diagnose** root cause by executing the test flow with Playwright
4. **Fix** broken steps using actual page state
5. **Apply** VIVIDUS story writing guidelines
6. **Update** story files with healed steps

---

## Step 1: Identify failing tests

Extract failure information from the user's input.

### Accepted inputs (at least one required)

| Input Type | Example |
|------------|---------|
| **Test run output** | Gradle/VIVIDUS error logs with stack traces |
| **Story file path** | `src/main/resources/story/web_app/Login.story` |
| **Scenario name** | "Test login" |
| **Error description** | "Login test fails because Submit button was renamed to Sign In" |
| **Test report** | Allure/VIVIDUS report JSON or summary |

### Failure detail extraction

From the input, extract:
1. **Failed step** — the exact VIVIDUS step that failed (e.g., `When I click on element located by \`buttonName(Submit)\``)
2. **Error type** — categorize the failure:
   - `ELEMENT_NOT_FOUND` — locator doesn't match any element
   - `TEXT_NOT_FOUND` — expected text not present on page
   - `TIMEOUT` — wait condition never satisfied
   - `STATE_MISMATCH` — element found but in wrong state (disabled, hidden)
   - `NAVIGATION_FAILURE` — page didn't load or URL changed
   - `STEP_SYNTAX_ERROR` — invalid step or parameter
3. **Context** — which page/state the test was in when it failed
4. **Story file** — path to the `.story` file containing the failure

**ABORT** if:
- No failure information is provided and no story file is specified
- The error relates to infrastructure (network, driver, server down) rather than test content

When aborting, explain what is needed and request test output or a story file path.

---

## Step 2: Locate the story file(s)

### Search strategy

1. **If story path is provided in error output** — read that file directly
2. **If scenario name is in the error** — search story files by scenario name
3. **If only a step text is provided** — grep for the failing step across all stories

### Search locations

- `src/main/resources/story/**/*.story` — all story files
- `src/main/resources/steps/**/*.steps` — composite steps (if failure is inside a composite)

### Read and understand context

- Read the **entire story file** to understand the full test flow
- Identify the navigation URL (from `Given I am on page with URL` or composite steps)
- Identify all steps leading up to the failure (the failure may be caused by an earlier broken step)
- Check if the story uses composite steps — if so, read the `.steps` file to understand the expanded flow

---

## Step 3: Diagnose root cause with Playwright

Use Playwright MCP to replay the test flow and identify what changed on the page.

### Diagnosis process

1. **Navigate** to the page URL from the story:
   - `browser_navigate(url)` — use the URL from the story's `Given I am on page with URL` step

2. **Replay steps leading to the failure**:
   - For each step before the failing step, perform the equivalent Playwright action
   - **DO NOT take screenshots**, use `browser_snapshot()` to capture page state at each significant step
   - Verify each step succeeds — if an earlier step fails, that's the real root cause

3. **At the failing step**:
   - Take a `browser_snapshot()` to see current page state
   - Search for the target element using the original locator strategy
   - If not found, search for the element by alternative strategies:
     - Text content (may have changed)
     - Position/structure (may have moved)
     - Nearby elements (use as anchors)
     - Similar attributes (partial matches)

4. **Document findings**:
   - What the story expected vs. what actually exists
   - The correct locator/text for the current page state
   - Any new elements that appeared (modals, banners, consent dialogs)
   - Any removed elements that the test depended on

### Root cause categories

| Category | Diagnosis Approach |
|----------|-------------------|
| **Renamed element** | Find element by role/position, note new text/label |
| **Changed ID/attribute** | Find element by visible text, note new ID |
| **Moved element** | Find element in new DOM position, update locator |
| **New blocking element** | Identify overlay/modal, add dismiss step |
| **Content text changed** | Find actual text, update assertion |
| **Page flow changed** | Map new navigation, add/remove wait steps |
| **Element removed** | Confirm removal, mark step for review |
| **Timing issue** | Check if element loads later, add/adjust wait |

### When to STOP and Ask (Do NOT Assume)

| Situation | Why Ask |
|-----------|---------|
| The page requires authentication not present in the story | Cannot access the page to diagnose |
| The entire page/application is down or returning errors | Infrastructure issue, not a test issue |
| The element appears to have been intentionally removed with no replacement | May indicate a feature removal — test may need deletion |
| Multiple equally valid fixes exist with different semantic meanings | Wrong fix could mask a real bug |
| The failure is inside a composite step used by many stories | Fix has cascading impact |
| The page behavior suggests a bug in the application, not a test issue | Should file a bug, not fix the test |

---

## Step 4: Fix broken steps

### Fix strategy by error type

#### ELEMENT_NOT_FOUND — Locator broken

1. Identify the element's new locator using the **Locator Stability Hierarchy** (Step 5)
2. Replace only the locator value, preserve the step structure:

```gherkin
!-- [HEALED] Element locator updated: buttonName(Submit) → buttonName(Sign In)
When I click on element located by `buttonName(Sign In)`
```

#### TEXT_NOT_FOUND — Content changed

1. Find the actual text on the page via Playwright snapshot
2. Update the text assertion:

```gherkin
!-- [HEALED] Page text changed: "Welcome back" → "Welcome back to your account"
Then text `Welcome back to your account` exists
```

#### TIMEOUT — Wait condition never satisfied

1. Check if the target element loads with a different locator
2. Check if the page flow changed (extra step needed before)
3. Fix the wait target or add a preceding action:

```gherkin
!-- [HEALED] Consent banner now appears before content loads — added dismiss step
When I click on element located by `buttonName(Accept Cookies)`
When I wait until element located by `caseInsensitiveText(Dashboard)` appears
```

#### STATE_MISMATCH — Element in wrong state

1. Verify if a prerequisite action is needed (scroll, click to enable, etc.)
2. Add the missing action:

```gherkin
!-- [HEALED] Checkbox must be checked before Submit becomes enabled
When I check checkbox located by `id(terms-agreement)`
When I click on element located by `buttonName(Submit)`
```

#### NAVIGATION_FAILURE — Page/URL changed

1. Check if URL structure changed
2. Update the URL or navigation step:

```gherkin
!-- [HEALED] URL path changed: /login → /auth/login
Given I am on page with URL `https://example.com/auth/login`
```

### Multi-step failures

When one broken step causes subsequent steps to also fail:
1. Fix the **root cause** (first failing step) only
2. Re-verify the downstream steps after the fix — they may now pass
3. Only fix downstream steps if they independently fail

### Discovery

VIVIDUS capabilities and project discovery:
1. **MUST** fetch available VIVIDUS steps via **VIVIDUS MCP**:
   - Call the MCP tool matching pattern `vividus_get_all_features`
   - **ABORT** if the VIVIDUS MCP tool is not available or not connected. Instruct the user to connect the VIVIDUS MCP server before proceeding. Without this tool, valid steps cannot be discovered and stories will contain incorrect syntax.
2. **MUST** read existing resources to learn patterns and conventions:
   - `src/main/resources/story/**/*.story` — existing stories
   - `src/main/resources/steps/*.steps` — reusable composite steps
3. **OPTIONAL**: obtain additional VIVIDUS documentation via **context7 MCP** when step syntax is unclear:
   - Call the MCP tool matching pattern `context7_resolve-library-id` with query `vividus` to get the VIVIDUS library ID
   - Call the MCP tool matching pattern `context7_query-docs` with the resolved library ID

⚠️ **Priority Rule:** Two sources of valid steps are allowed: (1) composite steps defined in project `.steps` files — these take precedence, and (2) steps returned by VIVIDUS MCP. If a composite step exists that accomplishes the same action as an MCP-returned step, always use the composite step.

**Strict rules:**
1. **ONLY use steps from project `.steps` files OR VIVIDUS MCP results** — NEVER invent steps
2. **Preserve exact syntax** — do not modify step parameters or structure
3. **If a required step is NOT available** — mark as `[MISSING STEP]`
4. **Preserve unchanged steps** — do NOT rewrite steps that are not broken

---

## Step 5: VIVIDUS Story Guidelines

### Locator Stability Hierarchy

When selecting a replacement locator, you **MUST** prefer in this order:

1.  🥇 **Exquisite**: `data-testid`, `data-test`, `data-qa`
2.  🥈 **High**: `id` (ONLY if human-readable and stable, e.g., `#submit-btn`. REJECT auto-generated IDs like `#ember123`)
3.  🥉 **Medium**: `buttonName()` or `linkText()` (Semantic and readable)
4.  ⚠️ **Low**: `caseInsensitiveText()` or `formName/fieldName` (Use with caution)
5.  ⛔ **Last Resort**: `cssSelector` or `xpath` (Only if NO other option exists. XPath must be robust)

### Key guidelines (apply during fix)

- **Avoid redundant verifications** — don't add a `Then text exists` after a wait that already confirms presence
- **Prefer buttonName for buttons** — use `buttonName(X)` over xpath for button elements
- **Synchronize after navigation** — if fix adds a new navigation, add a wait for the first interactive element on the new page
- **Use VIVIDUS expressions for dynamic values** — never hardcode dates, IDs, or generated data

### Expressions reference

- `#{randomInt($min, $max)}` — random integer
- `#{generateDate($period, $format)}` — date generation
- `#{generate($Provider.$method)}` — fake data (DataFaker)
- `#{anyOf($v1, $v2, $v3)}` — random pick from list

---

## Step 6: Update story files with healed steps

### Output

Edit the existing story file **in place**. Do NOT create new files.

### Healing rules

1. **Minimal changes** — fix ONLY the broken step(s). Do NOT rewrite working steps.
2. **Add `[HEALED]` comments** — mark every healed step with an inline comment explaining the fix:

```gherkin
!-- [HEALED] Locator updated: id(old-submit) → buttonName(Submit)
When I click on element located by `buttonName(Submit)`
```

3. **Preserve everything else**:
   - Meta tags unchanged
   - Scenario names unchanged
   - Working steps unchanged
   - Existing comments preserved

4. **Mark uncertain fixes** — if the fix is a best-guess:

```gherkin
!-- [HEALED][NEEDS REVIEW] Element not found — closest match used. Verify this is correct.
When I click on element located by `caseInsensitiveText(Sign In)`
```

5. **Mark unfixable steps** — if the element is genuinely gone with no replacement:

```gherkin
!-- [HEALED][REMOVED] Element no longer exists on page — step commented out for review
!-- When I click on element located by `id(deleted-feature-btn)`
```

### Composite step healing

If the failure is inside a composite step in a `.steps` file:
1. Identify all stories using the composite step
2. **Warn the user** about cascading impact before fixing
3. Only fix after explicit user confirmation
4. If only one story is affected, consider inlining the fix in the story instead

### Summary

After healing, provide:
- **File(s) modified**: paths of changed files
- **Failures fixed**: list of broken steps and their fixes
- **Root cause**: what changed in the application
- **Confidence level**: HIGH (verified with Playwright) / MEDIUM (best guess) / LOW (needs review)
- **Impact**: other stories potentially affected by the same change
- **Recommendation**: if the same locator pattern is used elsewhere, suggest proactive fixes
