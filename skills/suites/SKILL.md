# Creating a Suite

A suite is a smart folder containing tests.

```
CreateSuite({
  accountId: "<id>",
  name: "My Suite",
  sensitiveDataSetting: "disabled",  // required — use "disabled", "fail_when_detected", or "pass_when_detected"
  suiteType: "web"                   // "web" or "mobile_app"
})
```

- `sensitiveDataSetting` is **mandatory** — omitting it causes a validation error.