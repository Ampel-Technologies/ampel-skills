# TPF And Test Steps

## Adding A Step

Use this checklist for a product app such as `test-app-amd`:

1. Create a class under the app's `tests/` folder.
2. Inherit from `FTTestStep<ProductTest>`, for example `FTTestStep<AmdTest>`.
3. Implement `GetOrderTestStep()`, `IncludedInsideTpf()`, `VisualizerName()`, and `Run()`.
4. Add a matching enum value to the product step-order enum.
5. Add a matching `TPF.json` step key when `IncludedInsideTpf()` returns `true`.
6. Use `SharedResources` for cross-step state, events, and clients.
7. Use the inherited `Assert` property for expectations.

Minimal shape:

```csharp
internal class CheckThing : FTTestStep<ProductTest>
{
    public override int GetOrderTestStep() => (int)ProductTestStepOrder.CheckThing;
    public override bool IncludedInsideTpf() => true;
    public override string VisualizerName() => "CheckThing";

    public override async Task Run()
    {
        Assert.IsNotNull(SharedResources.Device, userMsg: "Device is not initialized");
        await Task.CompletedTask;
    }
}
```

## TPF Matching Rules

For TPF-controlled steps:

- `VisualizerName()` must match the `TPF.json` key exactly.
- `IsEnabled: false` disables the full step.
- A step with modules is also skipped when all modules are disabled.
- `StepConfig` is assigned only for enabled, TPF-included steps.
- Missing TPF keys are logged and the step is not added.

For non-TPF steps, return `false` from `IncludedInsideTpf()` and do not rely on `StepConfig`.

## Measurement Config

The current framework model supports:

```json
{
  "CheckBattery": {
    "IsEnabled": true,
    "Modules": {
      "Step1": {
        "Measurements": {
          "VBAT": {
            "Expected": 4600,
            "Unit": "mV",
            "AllowedRange": [4200, 5000]
          }
        }
      }
    }
  }
}
```

Use `StepConfig!.Modules["Step1"].Measurements["VBAT"]` only after asserting `StepConfig` is not null and only when the TPF is expected to contain that module.

## Lifecycle

Step execution order is:

1. `SetTeardown()`
2. `RunSetup()`
3. status update to `Pending`
4. `Run()`
5. status update to `Passed` when no exception/assert failure occurs
6. `RunTeardown()` in `finally`

Use setup/teardown hooks for resources that must be opened/closed around one step. Keep long-running or cancellable hardware waits responsive where possible, because pause and stop take effect between or around step execution rather than forcibly interrupting arbitrary blocking I/O.

## Assertions

Prefer the inherited `Assert` property from `FTTestStep<TTest>`. It writes to the owning test's `FTAssert` result list.

Common methods:

- `Assert.IsTrue(...)` / `Assert.IsFalse(...)`
- `Assert.IsNotNull(...)` / `Assert.IsNull(...)`
- `Assert.AreEqual(expected, actual, unit, userMsg: "...")`
- `Assert.AreEqual(measurementExpectation, actual, unit, userMsg: "...")`

Avoid creating a new standalone `FTAssert` inside a step unless there is a deliberate reason; standalone assertions will not contribute to the shared test result in the same way.

