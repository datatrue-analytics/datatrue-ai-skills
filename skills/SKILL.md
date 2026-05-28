---
name: datatrue-ai-skills
description: >
  Best practices for using the DataTrue MCP integration to create, configure, run,
  and analyse tag validation tests. Use this skill whenever the user wants to work
  with DataTrue — including creating suites, tests, steps, tag validations, data
  layer validations, running tests, or reviewing results. Also trigger when the user
  mentions "tag auditing", "GA4 validation", "GTM dataLayer checks", "cookie consent
  testing", "marketing tag QA", or any reference to the DataTrue platform. Even if
  the user just says "check my tags" or "validate analytics", this skill should be
  consulted first.
---


# DataTrue MCP Skill

This skill captures hard-won best practices for using the DataTrue MCP tools to
build, run, and interpret tag validation tests. Following these patterns will save
significant trial-and-error time.

## Sub-skills
suites: ./suites/SKILL.md
personas: ./personas/SKILL.md
tests: ./tests/SKILL.md
test-results: ./test-results/SKILL.md

Before working with suites, fetch: {base}/suites/SKILL.md

## Quick Reference: Object Hierarchy

DataTrue uses a strict hierarchy. Always create objects in this order:

```
Account → Suite → Test → Steps → Validations (Tag + DataLayer)
```

1. **Account** — the top-level container. Use `ListAccounts` to find the account ID.
2. **Suite** — groups related tests. Requires `accountId`, `name`, and `sensitiveDataSetting`.
3. **Test** — a simulation, coverage, email, or mobile_app test inside a suite.
4. **Step** — an action inside a test (navigate, click, type, run script, etc.).
5. **Validations** — tag validations and data layer validations attached to steps.

## Data Model: Entity Relationships and ID Linkage

Understanding how configuration entities relate to result entities is essential for
correctly querying and interpreting test results.

### Entity Relationship Diagram

```
Account
  └── Suite
        └── Test
              └── Step  (position-ordered, 0-based)
                    ├── TagValidation
                    │     └── TagPropertyValidation  (property/operator/value rules)
                    └── DataLayerValidation

TestRun  (triggered via RunTest)
  └── TestResult  (one per Test, retrieved via ListTestResults)
        └── StepResult  (one per Step, accessed via results.steps[].stepResultId)
              ├── TagValidationResult  (pass/fail per TagValidation)
              │     └── TagValidationCapturedRequestResult  (actual URLs + payloads)
              └── DataLayerValidationResult  (pass/fail + captured value)
```

### Configuration → Results ID Linkage

This is the most important and non-obvious part. The IDs used in configuration objects
do NOT appear directly in result objects — instead they are referenced via "spec" objects
that snapshot the configuration at the time the test ran.

| Configuration | Config ID | Links to Result via |
|---|---|---|
| `Step.id` | e.g. `1155` | `results.steps[n].stepResultId` (e.g. `203`) |
| `TagValidation.id` | e.g. `415` | `TagValidationResult.tagValidationSpec.tagValidationId` |
| `TagPropertyValidation.id` | e.g. `1469` | rolled up into `TagValidationResult.status` — no per-property result |

## Creating a Test

```
CreateTest({
  name: "My Test",
  suiteId: "<suite_id>",
  testType: "simulation",  // "simulation", "coverage", "email", or "mobile_app"
  enabled: true
})
```

AFter create a new test share a link to it using the following pattern
```https://app.datatrue.com/tests/<testId>/steps```

## Creating Steps

### CRITICAL: Add pauses to allow slow tags to fire - especially GA4 for Firebase.

Tags (especially GA4 and firebase) need time to execute after
a page loads or after an interaction like clicking a consent button. **Always add a
`pause` of at least 8 seconds** to steps where you expect tags to fire. Without this,
tag validations will fail because the network requests haven't been captured yet.

```
CreateStep({
  testId: "<test_id>",
  name: "Go to Homepage",
  action: "goto_url",
  target: "https://example.com",
  position: 0,
  pause: 8          // ← ESSENTIAL: wait for tags to fire
})
```

### Step actions selectors

Steps that interact with a web page such as clicking on a link or button, entering text
into a text field or selecting from a list or radio button selection should always use 
reliable CSS created by inspecting the page using Chrome DevTools.  

When inspecting a site to get css selectors always start the browser in incognito mode 
so there is not persisted state.

Always restart Chrome DevTools when looking at a new website

### Cookie Consent Buttons

Cookie consent implementations vary widely across sites. **Never assume a selector —
always inspect the actual page first** using browser DevTools or the Chrome DevTools
MCP tools.

Always restart Chrome DevTools when looking for consent button selectors. Consent 
banners will not appear if consent has been previously granted.

Common consent platforms and their selectors:

| Platform   | Typical Selector                                          |
|------------|-----------------------------------------------------------|
| iubenda    | `.iubenda-cs-accept-btn`                                  |
| Cookiebot  | `#CybotCookiebotDialogBodyLevelButtonLevelOptinAllowAll`  |
| OneTrust   | `#onetrust-accept-btn-handler`                            |
| TrustArc   | `.call` or platform-specific                              |

**Best practice**: Use `selectorType: "css"` and prefer class-based or ID-based
selectors. Always verify by loading the page in the browser first.

### Step Actions Reference

Common actions for web simulation tests:

- `goto_url` — navigate to a URL (set `target` to the URL)
- `click_button` — click a button (set `selector` + `selectorType`)
- `click_link` — click a link
- `click_element` — click any element
- `text_field` — type into a field (set `selector` + `target` for the value)
- `run_script` — execute JavaScript (set `jsCode`)
- `scroll_to` — scroll to an element
- `hover` — hover over an element

## Tag Definitions

DataTrue maintains a library of tag definitions which define the criteria 
(hostname and pathanme regex) for detecting requests that belong to specific tags.

You will need to search for tag definitions using ListTagDefinitions when creating tag validations.  
It is best to search using the name field.

```
ListTagDefinitions({
  first: 10,
  where: { name: { like: "%<tag_name>%" } }
})
```

## Tag Validations

Tag validations check that specific network requests (tags) fire during a step.
They use regex patterns for hostname and pathname matching.

A tag validation must have a tag type which is selected from DataTrue tag definition 
library. The tag validation hostnameDetection, pathnameDetection, hostnameValidation and 
pathnameValidation fields should be populated from the Tag Definition fields 
hostnameRegex and pathnameRegex.

The hostnameValidation and pathnameValidation fields can be modified to refine 
the validation but the hostnameDetection and pathnameDetection should not be altered.

```
CreateTagValidation({
  name: "GA4 Tag Fires",
  parentId: "<step_id>",
  parentType: "step",
  tagDefinitionId: "<tag_definition_id>",
  hostnameDetection: "<tag_definition.hostnameRegex>",
  pathnameDetection: "<tag_definition.pathnameRegex>",
  hostnameValidation: "<tag_definition.hostnameRegex>",
  pathnameValidation: "<tag_definition.pathnameRegex>",
  validationEnabled: true,
  enabled: true
})
```

### Tag Property Validations

Tag validations have tag properties.  A tag property is used to validate individual 
key value pairs found the in tag payload. For example, an event name or event parameter.

### Key Gotchas

- **Detection vs Validation**: `hostnameDetection` and `pathnameDetection` control
  which requests are *captured*. `hostnameValidation` and `pathnameValidation` control
  what the captured requests are *validated against*. Usually these are the same.
- **GA4 hostnames**: GA4 can send to `google-analytics.com`, `analytics.google.com`,
  or `googletagmanager.com`. Use a broad regex covering all variants.
- **GA4 pathnames**: Include `/g/collect`, `/j/collect`, and `/gtag` to catch all
  GA4 request patterns (the gtag.js loader and the collection endpoints).
- **Escape dots in regex**: Always escape literal dots (`\\.`) in hostnames.

## Data Layer Validations

Data layer validations check JavaScript variables, cookies, DOM elements, or custom
JS on the page. There are three main parts to data layer validations, the data source, 
assignment of a value returned from a source to a variable and the validation of the 
returned value either as a string or a JSON object. 

### Read/View only data layer validations

A read only data layer validation has a source configuration that collects data, but 
not assignment to a variable or validation of its value/propeties.  This is useful if 
the user wants to capture the value of a data source on a particular step of a test. 
The GTM dataLayer variable is an array that gets updated after page interactions and 
hence have a read only data layer validation that displays the values on the array can 
be helpful.

Leave validationEnabled as false for read only data layer validations.

### CRITICAL: Type handling for dataLayer

The GTM `dataLayer` is a JavaScript **Array**, not a string. Page interaction event data 
are pushed onto the array when DataTrue interacts with a page.

Users will want to validate objects that have been pushed to the array.  The following 
code can be used with the js_variable source to return a specific event in the array.

```
dataLayer.find(e => e.event==='add_to_cart')
```

### Working Pattern for dataLayer Checks

```
CreateDataLayerValidation({
  name: "GTM dataLayer Contains gtm.js",
  stepId: "<step_id>",
  source: "custom_js",
  customJsCode: "return JSON.stringify(window.dataLayer)",
  validationEnabled: true,
  enabled: true,
  regex: "gtm\\.js",
  configuration: {
    propertyValidations: [{
      name: "value",
      operator: "contains",
      value: "gtm.js"
    }]
  }
})
```

### Validation Requirements

- When `validationEnabled: true`, you **must** provide `configuration.propertyValidations`
  with at least one entry. Without this, the API returns: *"Data layer validation is
  enabled but no validation data has been specified"*.
- Each property validation needs `name`, `operator`, and `value`.
- Valid operators include: `contains`, `equals`, `greater_than`, `less_than`, etc.
  But be careful with type mismatches (see above).
- If you only want to check *presence* (not content), set `validationEnabled: false`
  — but note this won't actually validate anything, it just captures the value.

### Data Layer Validation Sources

| Source       | Use Case                                    | Key Fields                    |
|--------------|---------------------------------------------|-------------------------------|
| `js_variable`| Read a JS variable directly                 | `jsVariableName`              |
| `custom_js`  | Run arbitrary JS and validate the result    | `customJsCode`                |
| `cookie`     | Read a cookie value                         | `cookieName`                  |
| `dom`        | Read a DOM element's attribute or text      | `selector`, `selectorType`, `attr` |
| `url`        | Validate the current URL                    | (uses regex)                  |

## Running Tests

DataTrue always runs tests with a new browser with non persisted state in cookies or application storage.

```
RunTest({
  id: "<test_id>",
  driverConfigurationId: "<browser_id>",  // optional — defaults to Chrome
  region: "<region_id>"                    // optional — defaults to available region
})
```

### Browser Selection

Use `ListBrowsers` to see available browsers. Common configurations:

- **Chrome** (ID varies) — the standard choice for web tests
- **Chrome (local)** — may have infrastructure issues; prefer the regular Chrome config
- **Chrome (No Proxy)** — useful when proxy interference is suspected
- **Firefox**, **Safari** — for cross-browser testing

**Tip**: If tests fail with `"session not created: Chrome instance exited"`, try a
different `driverConfigurationId`. This error is usually an infrastructure issue,
not a test configuration problem.

### Polling for Results

Tests run asynchronously. After calling `RunTest`, poll with `ListTestResults`:

```
ListTestResults({ first: 1, testId: "<test_id>" })
```

The `state` field progresses through:
`initialised` → `running` → `success` | `failed` | `error`

- **success**: All validations passed
- **failed**: Test completed but one or more validations failed
- **error**: Test could not complete (e.g., browser crash, element not found)

**Important**: Poll multiple times — tests typically take 20-60 seconds depending on
page complexity and pause durations.

## Interpreting Results

### Test Results Structure

The `results` object in `ListTestResults` contains:

- `steps[]` — one entry per step, each containing:
  - `error` — null if the step succeeded, error message if it failed
  - `responseTime` — page load time in milliseconds
  - `consoleLogs[]` — browser console output (useful for debugging)
  - `dataLayerValidations[]` — data layer validation results
  - `stepResultId` — use this to query tag validation results
- `auxData` — browser, region, and run metadata
- `averagePageLoadTime` — average across all steps

### Checking Tag Validation Results

Use `ListTagValidationResults` with the `stepResultId`:

```
ListTagValidationResults({
  first: 10,
  where: { stepResultId: { eq: "<step_result_id>" } }
})
```

To see captured requests for a specific tag validation:

```
ListTagValidationCaptureRequestResults({
  first: 10,
  where: { tagValidationResultId: { eq: "<tag_validation_result_id>" } }
})
```

This reveals the actual URLs and payloads captured, which is invaluable for debugging.

## Common Workflows

### Discover what tags are on a site - Tag Audit

1. Create a new suite called "Tag Audit"
2. Create a test called "Tag Audit"
3. Create a coverage step called "Scan website" that scans from the home page of the web site for 10 pages with a page depth of 50 and a pause of 15 seconds

If the site has a consent manager, then add steps before the coverage step to load the home page
and accept all cookies on the consent manager banner

### Validate GA4 + GTM on a Website

See `references/ga4-gtm-workflow.md` for a complete step-by-step workflow.

### Debug a Failing Tag Validation

1. Check the step's `error` field — did the step itself fail?
2. Check `consoleLogs` for JavaScript errors that might block tags
3. Use `ListTagValidationCaptureRequestResults` to see what WAS captured
4. Broaden hostname/pathname regex patterns if needed
5. Increase the step `pause` if tags fire late

### Debug a Failing Data Layer Validation

1. Check `errorClass` and `errorMessage` in the validation result
2. `TypeError` usually means a type mismatch (Array vs String)
3. Switch to `source: "custom_js"` with `JSON.stringify()` for complex objects
4. Check that `configuration.propertyValidations` operators match the data type

## API Quirks and Pitfalls

1. **Deleting validations** — use `DeleteTagValidations` or `DeleteDataLayerValidations`
   with a `where` filter (e.g., `{ id: { eq: "<id>" } }`).
2. **Position field** — steps use 0-based positioning. If omitted, the step is
   appended at the end.
3. **Tag validations can be on steps OR tests** — use `parentType: "step"` for
   step-level or `parentType: "test"` for test-level validations. Step-level is
  usually more precise.
4. **Region availability** — use `ListRegions` to check available regions before
   specifying one. There may only be one available.