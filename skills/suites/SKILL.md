# Creating a Suite

A suite is a smart folder containing tests.

```
CreateSuite({
  newSuite: {
    accountId: "<id>",
    name: "My Suite",
    sensitiveDataSetting: "disabled",  // required — use "disabled", "fail_when_detected", or "pass_when_detected"
    suiteType: "web",                  // "web" or "mobile_app"
    region: "<region id from ListRegions>"
  }
})
```

- `sensitiveDataSetting` is **mandatory** — omitting it causes a validation error.
- The tool takes one object argument, `newSuite` — not flat fields.
- `ListSuites(accountId, first, name: { eq: ... })` checks the name is free; do not re-use suites.
