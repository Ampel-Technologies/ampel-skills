# Architecture

## Repository Shape

FT product repositories are .NET 8 Windows workspaces with one product app and several Git submodules. The `ft-amd` reference project uses:

- `test-app-amd`: product-specific executable, resources, firmware/label assets, `TPF.json`, and runtime config.
- `ft-test-framework`: generic functional-test execution framework.
- `ft-gui`: WPF GUI library and operator interaction surface.
- `ft-logger`: Serilog-backed FT logging library.
- `ft-database-client`: production database client and activity models.
- `ft-label-printer`: label printing library.
- `ft-login-client`: Microsoft identity login client.

Use `.gitmodules` and the solution file to confirm exact submodule paths and pinned commits before changing code.

## Framework Core

The current framework pattern is:

- `FTTest<TTest>` owns shared resources, test result state, assertion collection, and lifecycle hooks.
- `FTTestStep<TTest>` owns one executable step and receives the product test aggregate through `SharedResources`.
- `FTTestStepManager<TTest>` discovers concrete step classes from loaded assemblies, filters them through TPF settings, attaches shared resources, and sorts by `GetOrderTestStep()`.
- `FTTestExecutor<TTest>` runs global setup, per-step setup/run/teardown, global teardown, pause, stop, and result finalization.
- `TestRunner` registers one or more `IFTTest` instances and exposes start, pause, stop, status, and unregister operations.

Primary framework files live under `ft-test-framework/TestFramework/Api` and `ft-test-framework/TestFramework/Core`.

## Execution Flow

1. The product app creates a `TaEvents` or equivalent product event bridge.
2. The product app constructs its product test class, usually `ProductTest : FTTest<ProductTest>`.
3. The product test constructor calls `base("TPF.json")`.
4. `FTTest` deserializes the TPF and asks the step manager to create steps.
5. The step manager discovers concrete `FTTestStep<ProductTest>` classes in loaded assemblies.
6. Steps included in TPF are matched by `VisualizerName()` to `TPF.json` keys.
7. Enabled steps are sorted by `GetOrderTestStep()` and exposed to the GUI/status layer.
8. `TestRunner.StartAllTestsAsync()` or `StartSingleTestAsync()` runs steps through the executor.

## Ownership Boundaries

- Put product constants, product state, hardware commands, and order-specific behavior in the product app.
- Put reusable execution, status, assertion, TPF parsing, and lifecycle behavior in `ft-test-framework`.
- Put generic operator UI primitives in `ft-gui`, not in each product app.
- Put database wire shape and activity model translation in `ft-database-client`.
- Put authentication flow in `ft-login-client`.
- Put logging setup and sinks in `ft-logger`.

Do not move product-specific hardware assumptions into shared libraries unless at least two product apps need the same behavior.

