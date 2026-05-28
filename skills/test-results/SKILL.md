### Tool Call Sequence to Report Results

Always follow this sequence to get the results of a test.:

```
Step 1: ListTestRuns(first: 1, testId: "<testId>")
        → get testRun.id

Step 2: ListTestResults(first: 1, testRunId: "<testRunId>")
        → get testResult.id
        → get results.steps[n].stepResultId  ← critical linking ID
        → get results.steps[n].error         ← did the step itself fail?
        → get state (success/failed/error)   ← overall test result
        → get results.dataLayerValidations[] ← data layer validation results
        → get results.steps[n].consoleLogs[] ← browser console logs for debugging

Step 3: ListDetectedSignals(first: 10, where: { stepResultId: { eq: "<stepResultId>" } })
        → get detected signals for the step
        → get persona items and captured request URLs for any signals found (e.g. email address leakage)

Step 4: ListTagAccounts(first: 10)
        → get tag accounts for the step (e.g. Google Analytics, Facebook Pixel)        
    
Step 5: ListTagValidationResults(first: 10, where: { stepResultId: { eq: "<stepResultId>" } })
        → get tagValidationResult.id
        → get tagValidationResult.status     ← "passed" or "failed"
        → get tagValidationSpec              ← tag validation config snapshot at time of test run

Step 6: ListTagValidationCaptureRequestResults(
          first: 10,
          where: { tagValidationResultId: { eq: "<tagValidationResultId>" } }
        )
        → get tagValidationCapturedRequestResultid
        → get the actual captured request URLs and payloads for debugging

Step 7: ListTagPropertyValidationResults(
          first: 10,
          where: { tagValidationCapturedRequestResultId: { eq: "<tagValidationCapturedRequestResultId>" } }
        )
        → get pass/fail status
        → get renderedValue
        → get tagPropertyValidationSpec (property/operator/value rules)
```

### Annotated Real Example

Below is a real example from test 177 (Readings.com.au GA4 validation) showing
how configuration IDs map through to results.

**Configuration:**
```
Test 177
  └── Step 1155  "Go to Readings Homepage"
        └── TagValidation 415  "GA4 - Homepage"
              ├── TagPropertyValidation 1469  { en equals "page_view" }
              └── TagPropertyValidation 1466  { tid equals "G-FTC8QEF4W1" }
```

**Test Results:**

A test result is generated whenever a test is run. A test can have many test results created. 
Ignore aborted test results unless asked not to.

```
TestResult 920  (state: "validated")
  └── StepResult 203  (stepResultId: "203", responseTime: 20941ms, error: null)
        └── TagValidationResult 224  (status: "passed")
              tagValidationSpec.name: "GA4 - Homepage"
              tagValidationSpec.tagValidationId: "415"  ← links back to TagValidation 415
```

You can create a link to the DataTrue UI test results using this url pattern
https://app.datatrue.com/tests/<testId>/results?result=<testResultId>


### Summarising Results: Recommended Display Format

When summarising test results, use this structure for each step:

```
### Step <position> — <step.name>
Response time: Xms | Error: <error or "none">

Tag Validation: <tagValidation.name> → ✅ passed / ❌ failed
| Property | Operator | Expected        | Result |
|----------|----------|-----------------|--------|
| en       | equals   | page_view       | ✅     |
| tid      | equals   | G-FTC8QEF4W1   | ✅     |
```

### Showing Per-Property Pass/Fail on a Failed Tag Validation

The API does not return per-property results — `TagValidationResult.status` is a single
pass/fail for the whole tag validation. To determine which individual property validations
passed or failed you must:

1. Call `ListTagValidationCapturedRequestResults` to get the actual captured request payloads
2. Compare each `TagPropertyValidation` rule against the captured payload manually
3. Mark each property as ✅ or ❌ based on whether the captured value satisfies the rule

**Tool call sequence for a failed tag validation:**

```
Step 1: ListTagValidationResults(where: { stepResultId: { eq: "<stepResultId>" } })
        → get tagValidationResult.id and status = "failed"

Step 2: ListTagValidationCapturedRequestResults(
          where: { tagValidationResultId: { eq: "<tagValidationResultId>" } }
        )
        → get the actual captured request URL and payload parameters

Step 3: ListTagPropertyValidations(first: 20, tagValidationId: "<tagValidationId>")
        → get the expected property rules

Step 4: Compare each rule against the captured payload to determine per-property pass/fail
```

**Example of a failed tag validation display:**

```
Tag Validation: GA4 - Homepage → ❌ failed

Captured request: https://www.google-analytics.com/g/collect?en=page_view&tid=G-XXXXXXXX

| Property | Operator | Expected       | Captured      | Result |
|----------|----------|----------------|---------------|--------|
| en       | equals   | page_view      | page_view     | ✅     |
| tid      | equals   | G-FTC8QEF4W1  | G-XXXXXXXX    | ❌     |
```

**Note**: If no request was captured at all (empty results from
`ListTagValidationCapturedRequestResults`), the tag did not fire — report this
as "no matching request captured" rather than a per-property failure.

### Loading the ListTagPropertyValidations Tool

`ListTagPropertyValidations` is a **deferred tool** and may not load with generic
search terms. If it doesn't appear, search explicitly:

```
tool_search("ListTagPropertyValidations tagValidationId")
```

This tool takes `tagValidationId` (the config ID, e.g. `"415"`) and returns all
property validation rules. It is essential for showing what was being checked,
since results only return pass/fail at the TagValidation level.
