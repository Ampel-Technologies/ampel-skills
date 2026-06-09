# ft-amd Reference Project Pattern

Use `ft-amd` as the concrete example of a product app built on the framework. Inspect the live repo before copying details; this file captures the observed structure from the current reference checkout.

## Product App Files

Important `test-app-amd` files:

- `Program.cs`: wires `TaEvents`, initializes the product test application, subscribes GUI events, and starts the GUI.
- `MainFTLogic.cs`: wraps `TestRunner` for app-level initialize, start, pause, stop, status, and user-input completion.
- `AmdTest.cs`: product aggregate extending `FTTest<AmdTest>`, with shared state, database client, lifecycle overrides, and status update hooks.
- `StepOrder.cs`: enum used by steps for deterministic order.
- `TPF.json`: runtime test procedure configuration.
- `TAInterface.cs`: GUI callbacks and operator dialogs.
- `TaEvents.cs`: app-specific event bridge and `TaskCompletionSource` user-input handoff.
- `GuiInterface.cs`: start, stop, pause, and close handlers.
- `tests/*.cs`: concrete `FTTestStep<AmdTest>` implementations.
- `Resources/*`, label templates, and firmware files: runtime assets copied by the app `.csproj`.

## Initialization Pattern

The app initializes by:

1. Creating `TaEvents` with callbacks to `TAInterface`.
2. Subscribing test-step status events to GUI color/status callbacks.
3. Creating `AmdTest` with events and database activity state.
4. Wiring login token events to the database client.
5. Registering the product test with `TestRunner.RegisterTest(...)`.
6. Querying test step names from `TestRunner.GetTestStatuses()`.
7. Starting the GUI with product title, logo, and step list.

Keep this bridge product-specific. It coordinates framework, GUI, login, and database concerns for the product app.

## Step Pattern

AMD steps generally:

- inherit from `FTTestStep<AmdTest>`
- return an `AmdTestStepOrder` enum value
- return `true` from `IncludedInsideTpf()`
- return a TPF key from `VisualizerName()`
- read or update `SharedResources.amdDevice`, `SharedResources.operation`, `SharedResources.LocalEvents`, or `AmdTest.dbClient`
- use `FTLogger` for production-line traceability
- use `Assert` for conditions that should fail the step

When adding a new product step, update all of: class, enum, TPF key, GUI/operator resources if needed, and packaging assets in the `.csproj` if the step requires new files.

## Operator Input

Operator prompts use a two-sided bridge:

- A step calls methods on `SharedResources.LocalEvents`, such as requesting barcode, text, label confirmation, assembly confirmation, enclosure confirmation, or LED confirmation.
- `TaEvents` creates a `TaskCompletionSource`.
- `TAInterface` opens or updates the GUI.
- `TestApplication.ProvideUserInput...` completes the task source.

Keep prompt text, images, and completion buttons in the product app unless they become reusable GUI widgets.

## Sensitive Runtime Config

`appsettings.json` may contain login settings, API keys, database hosts, product tags, logging paths, and debugger serials. Never copy real values into examples or documentation. Use placeholders when documenting shape.

