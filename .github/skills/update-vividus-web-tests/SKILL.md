---
name: update-vividus-web-tests
description: 'Find and update existing VIVIDUS test automation stories based on a provided scenario or change description. Modifies .story files following VIVIDUS syntax and project conventions. Use when: updating existing web tests, modifying test steps after UI changes, fixing broken locators, adding new steps to existing scenarios.'
argument-hint: 'Describe what changed or provide the updated scenario...'
---

## Process Overview

1. **Retrieve** update scenario from user input
2. **Find** existing story files that match the scenario
3. **Execute** updated scenario with Playwright
4. **Analyze** changes and map each action to a VIVIDUS step
5. **Apply** VIVIDUS story writing guidelines
6. **Update** existing VIVIDUS stories

---

## Step 1: Retrieve update scenario

Extract the update scenario from the user's prompt input.

The input **must** contain at least one of:
- **Change description**: What changed in the application or test (e.g., "button text changed from Save to Submit", "new confirmation modal added after form submission")
- **Updated test steps**: Revised numbered sequential actions reflecting the new behavior
- **Expected results**: Updated verifiable outcomes
- **Story file reference**: Path or name of the story to update

**ABORT** further execution if:
- No change description AND no updated test steps are provided
- The described change is too vague to identify which story to update (e.g., "fix the tests")
- Input does not describe a testable change

When aborting, explain what is missing and request a clear change description or updated test steps.

---

## Step 2: Find existing story files

Locate the story file(s) that need to be updated.

### Search strategy

1. **If a story path/name is provided** — read that file directly
2. **If a scenario name or test case ID is provided** — search story files by:
   - `@testCaseId` meta tag
   - Scenario name text
   - Feature name in meta tags
3. **If only a change description is provided** — search by:
   - Related step text patterns (e.g., button names, URL patterns, page elements mentioned in the change)
   - Feature folder structure matching the described area

### Search locations

- `src/main/resources/story/**/*.story` — all story files
- `src/main/resources/steps/**/*.steps` — composite steps (if a composite step needs updating)

### Validation

- **ABORT** if no matching story file is found — inform the user and ask for clarification
- **Confirm with user** if multiple candidate files are found and the correct one is ambiguous
- Read and display the current content of the identified story file before proposing changes

---

## Step 3: Execute updated scenario with Playwright

Use Playwright MCP to validate the updated scenario and collect element locators.

### Execution process

1. **Navigate**: `browser_navigate(url)` — URL from the existing story or user prompt

2. **For each updated test step**:
   - **DO NOT take screenshots**, use `browser_snapshot()` to take a page snapshot to understand page structure and its elements
   - Identify changed elements: new/renamed buttons, relocated fields, added modals, removed elements
   - Collect stable locator attributes: IDs, data-testid, aria-labels, exact button/link text
   - Perform actions to verify element behaviors: `browser_click`, `browser_type`, `browser_select_option`, or `browser_run_code`
   - Document differences between the existing story and actual page state

3. **Dynamic content**: `browser_wait_for(text)` for async operations

4. **Compare with existing story**: Identify which steps are still valid, which need updating, and which are new

### Assumption Handling

When encountering unclear changes, the agent should:
1. Proceed with reasonable assumption or workaround
2. Document assumption or workaround clearly
3. Flag with an `[ASSUMPTION]` inline comment in the updated story file

Example assumptions:

| Situation | Assumption Made |
|-----------|-----------------|
| Button renamed but TC doesn't specify new text | Used actual text from app exploration |
| New element appeared between existing steps | Added wait/interaction step at appropriate position |
| Element locator broken but element still exists | Updated locator using stability hierarchy |
| Step removed from TC but present in story | Commented out with note for review |

### When to STOP and Ask (Do NOT Assume)

Do **NOT** proceed with assumptions in these situations. Stop execution and request clarification:

| Situation | Why Ask |
|-----------|---------|
| Authentication credentials required but not provided | Security-sensitive, cannot guess |
| Target environment URL missing or unclear | Wrong environment could cause data issues |
| The entire test flow has fundamentally changed | Cannot determine which parts to preserve |
| Multiple scenarios in a story could match the change | Updating the wrong scenario would break tests |
| Change requires deleting a scenario entirely | Destructive action needs explicit confirmation |
| Change affects composite steps used by multiple stories | Cascading impact needs user awareness |

---

## Step 4: Analyze changes and map each action to a VIVIDUS step

### Change Classification

Before modifying anything, classify each change:

| Change Type | Action |
|-------------|--------|
| **Locator update** | Replace locator value only, preserve step structure |
| **Step text update** | Replace expected text/value in existing step |
| **Step addition** | Insert new step(s) at correct position |
| **Step removal** | Remove step (with inline comment explaining why) |
| **Step reorder** | Move step to new position in sequence |
| **Wait/sync addition** | Add synchronization after new navigation points |
| **Meta tag update** | Update meta information (feature, priority, etc.) |

### Logic & Flow Planning

**Before choosing any steps**, write out the logical flow of the updated test:
1. Identify which *high-level* actions changed vs. remained the same
2. Ensure the updated sequence handles new state changes (e.g., "New modal must be dismissed before proceeding")
3. Verify wait/synchronization steps are still placed correctly after navigation changes
4. Check that verifications still target the correct elements/text

### Discovery

VIVIDUS capabilities and project discovery:
1. **MUST** fetch available VIVIDUS steps via **VIVIDUS MCP**:
   - Call the MCP tool matching pattern `vividus_get_all_features`
   - **ABORT** if the VIVIDUS MCP tool is not available or not connected. Instruct the user to connect the VIVIDUS MCP server before proceeding. Without this tool, valid steps cannot be discovered and stories will contain incorrect syntax.
2. **MUST** read existing resources to learn patterns and conventions:
   - `src/main/resources/story/**/*.story` — existing stories
   - `src/main/resources/steps/*.steps` — reusable composite steps
3. **MUST** review Lifecycle and Examples usage (transformers, data tables), scenario structure and naming, meta tags
4. **OPTIONAL**: obtain additional VIVIDUS documentation via **context7 MCP** when step syntax is unclear or additional capabilities need to be explored:
   - Call the MCP tool matching pattern `context7_resolve-library-id` with query `vividus` to get the VIVIDUS library ID
   - Call the MCP tool matching pattern `context7_query-docs` with the resolved library ID to look up specific step documentation, configuration options, or plugin capabilities

⚠️ **Priority Rule:** Two sources of valid steps are allowed: (1) composite steps defined in project `.steps` files — these take precedence, and (2) steps returned by VIVIDUS MCP. If a composite step exists that accomplishes the same action as an MCP-returned step, always use the composite step.

**Strict** rules to adhere:
1. **ONLY use steps from project `.steps` files OR VIVIDUS MCP results** — NEVER invent, modify, or assume steps that are not explicitly listed in either source
2. **Preserve exact syntax** — do not modify step parameters or structure
3. **If a required step is NOT available in either MCP results or project `.steps` files** — DO NOT silently ignore, mark as `[MISSING STEP]`
4. **Preserve unchanged steps** — do NOT rewrite steps that are not affected by the change

## Step 5: VIVIDUS Story Guidelines

### General rules

1. **Locators:** Follow the **Locator Stability Hierarchy** below to ensure stability.
2. **Expressions:** Use VIVIDUS expressions instead of hardcoded dynamic values — see **Use VIVIDUS Expressions Instead of Hardcoded Values** below.
3. **Data Tables:** Use Examples blocks **only** when a scenario must run with multiple distinct data sets. Do NOT use Examples for a single data set — inline values or expressions directly.
4. **Composite Steps:** Propose new composite steps for repeated action patterns.
5. **Contextual Steps:** When using parent element context, ensure child locators are relative.

### Use VIVIDUS Expressions Instead of Hardcoded Values

NEVER hardcode dynamic values (dates, IDs, names, emails, random data). Use VIVIDUS expressions instead so stories remain re-executable without manual edits.

Key built-in expressions:
- **Random integer**: `#{randomInt($minInclusive, $maxInclusive)}` — e.g. `#{randomInt(1000, 9999)}`
- **Generate date**: `#{generateDate($period, $outputFormat)}` — e.g. `#{generateDate(P, yyyy-MM-dd)}` (today), `#{generateDate(P1D, yyyy-MM-dd)}` (tomorrow)
- **Fake data** (via DataFaker): `#{generate($Provider.$method)}` — e.g. `#{generate(Name.firstName)}`, `#{generate(Internet.emailAddress)}`, `#{generate(Address.fullAddress)}`
- **Pick random from list**: `#{anyOf($value1, $value2, $value3)}`
- **String transforms**: `#{toLowerCase($input)}`, `#{toUpperCase($input)}`

❌ **Bad** — hardcoded values:

```gherkin
When I enter `John` in field located by `xpath(//input[@name='firstName'])`
When I enter `john.doe@test.com` in field located by `xpath(//input[@name='email'])`
When I enter `2026-04-24` in field located by `xpath(//input[@name='date'])`
```

✅ **Good** — generated values:

```gherkin
When I enter `#{generate(Name.firstName)}` in field located by `xpath(//input[@name='firstName'])`
When I enter `#{generate(Internet.emailAddress)}` in field located by `xpath(//input[@name='email'])`
When I enter `#{generateDate(P, yyyy-MM-dd)}` in field located by `xpath(//input[@name='date'])`
```

### Locator Stability Hierarchy

When identifying elements, you **MUST** prefer locators in this order:

1.  🥇 **Exquisite**: `data-testid`, `data-test`, `data-qa`
2.  🥈 **High**: `id` (ONLY if it looks human-readable and stable, e.g., `#submit-btn`. REJECT auto-generated IDs like `#ember123`)
3.  🥉 **Medium**: `buttonName()` or `linkText()` (Semantic and readable)
4.  ⚠️ **Low**: `caseInsensitiveText()` or `formName/fieldName` (Use with caution for localization)
5.  ⛔ **Last Resort**: `cssSelector` or `xpath` (Only if NO other option exists. XPath must be robust, avoiding indexing like `div[3]/span[2]`)

### Avoid Redundant Verifications

Do NOT verify the same element/text twice. If you wait for an element, it's already verified.

❌ **Bad** — redundant check:

```gherkin
When I wait until element located by `caseInsensitiveText(My Account)` appears
Then text `My Account` exists
```

✅ **Good** — single verification:

```gherkin
When I wait until element located by `caseInsensitiveText(My Account)` appears
```

### Prefer buttonName Locator for Buttons

When interacting with button HTML elements, use `buttonName` locator instead of xpath.

❌ **Bad** — verbose xpath:

```gherkin
When I click on element located by `xpath(//button[contains(text(),'Save')])`
```

✅ **Good** — clean buttonName locator:

```gherkin
When I click on element located by `buttonName(Save)`
```

### Synchronize After Navigation

**CRITICAL RULE**: When navigating to a new page or opening a new tab, **ALWAYS** add a wait step for FIRST **interactive element** on that page or tab. This ensures the page has fully loaded and all subsequent interactive elements are available.

**Why**: Waiting for the first interactive element on a page guarantees that:
- The page DOM is fully rendered
- JavaScript has executed and initialized components
- All form fields, buttons, and other interactive elements are ready
- Subsequent steps won't fail due to elements not being available yet

✅ **Good** — wait for first interactive element after navigation:

```gherkin
When I click on element located by `buttonName(Add Product)`
When I wait until element located by `caseInsensitiveText(Create Product)` appears

!-- Now safe to interact with form fields without additional waits
When I enter `${campaignName}` in field located by `xpath(//input[@name='name'])`
When I enter `${campaignName}` in field located by `xpath(//input[@placeholder='URL'])`
```

❌ **Bad** — no synchronization after navigation:

```gherkin
When I click on element located by `buttonName(Add Product)`
!-- Missing wait — next step may fail if page hasn't loaded
When I enter `${campaignName}` in field located by `xpath(//input[@name='name'])`
```

❌ **Bad** — waiting before every field (unnecessary):

```gherkin
When I click on element located by `buttonName(Create Product)`
When I wait until element located by `xpath(//input[@name='name'])` appears
When I enter `${campaignName}` in field located by `xpath(//input[@name='name'])`
When I wait until element located by `xpath(//input[@placeholder='URL'])` appears
When I enter `${campaignName}` in field located by `xpath(//input[@placeholder='URL'])`
```

**When to wait:**
- ✅ After clicking navigation links (new page loads)
- ✅ After clicking buttons that open new tabs or modals
- ✅ After dropdown selections that dynamically load/show new fields
- ✅ After form submissions that redirect to different pages
- ❌ Before every field on the same page (only first element needed)
- ❌ Between consecutive actions on already-loaded elements

## Step 6: Update existing VIVIDUS stories

### Output

Edit the existing story file **in place**. Do NOT create a new file unless the existing file cannot be found.

### Update rules

1. **Minimal changes** — only modify steps affected by the change. Do NOT rewrite the entire file.
2. **Preserve meta tags** — keep existing `@testCaseId`, `@requirementId`, `@feature`, `@priority` unless the user explicitly requests a change.
3. **Preserve scenario names** — keep existing scenario names unless the change fundamentally alters the scenario's purpose.
4. **Preserve comments** — keep existing inline comments (`!--`) unless they are no longer relevant.
5. **Add change comments** — mark updated sections with an inline comment explaining the change:

```gherkin
!-- [UPDATED] Button text changed from "Save" to "Submit" per UI redesign
When I click on element located by `buttonName(Submit)`
```

6. **Mark assumptions** — flag any assumptions made during the update:

```gherkin
!-- [ASSUMPTION] New modal appears after save — added wait for confirmation text
When I wait until element located by `caseInsensitiveText(Changes saved successfully)` appears
```

7. **Mark missing steps** — if a required step doesn't exist:

```gherkin
!-- [MISSING STEP] Need step to verify toast notification disappears after 3 seconds
```

### Composite step updates

If the change affects a composite step in a `.steps` file:
1. Identify all stories that use the composite step
2. Warn the user about cascading impact
3. Update the composite step only after user confirmation
4. List affected stories in a comment within the `.steps` file

### Summary

After updating, provide a brief summary:
- **File(s) modified**: paths of changed files
- **Changes made**: list of specific modifications
- **Assumptions**: any assumptions that need validation
- **Impact**: other stories or composite steps potentially affected
