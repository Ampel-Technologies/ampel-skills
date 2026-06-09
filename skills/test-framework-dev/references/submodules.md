# Submodules

Product FT apps use Git submodules for shared libraries. Always check `.gitmodules`, submodule status, and each submodule's own git status before editing.

## ft-test-framework

Purpose: generic FT execution framework.

Look here for:

- `FTTest<TTest>` product aggregate behavior
- `FTTestStep<TTest>` step lifecycle
- `TestRunner` registration and control
- TPF records and measurement expectations
- assertion/result behavior
- executor, pause/stop, and status update behavior

Prefer framework changes only when behavior should apply to all product apps.

## ft-gui

Purpose: WPF operator GUI library.

Look here for:

- `GuiApi` start/stop/pause/close events
- test-step display and color updates
- barcode and text input popups
- operator instruction panels
- startup config dialogs
- GUI log sinks or telemetry display

Product-specific images and wording usually belong in the product app, not `ft-gui`.

## ft-logger

Purpose: shared logging backed by Serilog.

Look here for:

- `FTLogger` methods and log levels
- file, console, and GUI sink behavior
- logger configuration loading
- log file naming or output directory changes

Use structured logging values for product IDs, serials, operations, and measured values.

## ft-database-client

Purpose: database API access and activity models.

Look here for:

- `DatabaseClient`
- `DbActivities`, `DbActivity`, `DbActivityOperation`, `DbActivityStatus`
- auth-token integration via events
- database request/response mapping

Keep database hostnames and auth details in runtime configuration. Do not hardcode production endpoints in code examples.

## ft-login-client

Purpose: Microsoft identity login and token flow.

Look here for:

- `LoginClient`
- token update events
- appsettings reading and settings builder classes
- silent login behavior

When the database client needs auth, wire login token updates in the product app initialization layer.

## ft-label-printer

Purpose: label printer integration.

Look here for:

- `FTAmpelLabelPrinter`
- printer settings model
- printer status model
- label template handling

Keep product-specific `.prn` label templates in the product app and ensure the app `.csproj` copies them to output.

## Submodule Edit Discipline

- If editing a submodule from inside a product repo, remember it is a nested git repo with its own branch, commit, and remote.
- Commit submodule changes in the submodule repo first, then update the product repo's pinned submodule commit.
- Do not mix unrelated product app and shared library changes unless the integration requires it.
- Validate both the submodule build/tests and the consuming product app build when public behavior changes.

