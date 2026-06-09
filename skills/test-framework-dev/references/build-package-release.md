# Build, Package, And Release

## Product Repo Build Pattern

FT product repos use a root `Makefile` around the main product `.csproj`. In `ft-amd`, the important settings are:

- `FT_NAME`: package and output prefix.
- `PROJECT`: main product app project, for example `test-app-amd/TestAppAmd.csproj`.
- `FRAMEWORK`: `net8.0-windows`.
- `RUNTIME`: `win-x64`.
- `OUTPUT_DIR`: staged packages.
- `ARTIFACTS_DIR`: build and publish artifacts.

Use `make help` first when available.

## Common Commands

From the product repo root:

```powershell
make restore
make build-debug
make run
make check
make build-release
make publish-release
make package-release
```

Use narrower commands during development:

- Product app compile check: `dotnet build <app.csproj> -c Debug`
- Framework tests: `dotnet test ft-test-framework/TestFramework.Tests/TestFramework.Tests.csproj`
- Release validation: `make check` then `make package-release`

## Versioning

The package version comes from `<Version>` in the main product `.csproj`, unless overridden by `make VERSION=...`.

Release builds require semantic version format `X.Y.Z`. The Makefile writes `versions.txt` into staged output and includes versions for referenced projects where available.

## Runtime Assets

Product app `.csproj` files usually copy these to output:

- `TPF.json`
- `appsettings.json`
- `Resources/*`
- label templates such as `.prn`
- firmware or hex files

When adding runtime files, update the app `.csproj` copy rules and test with `make run` or a publish target from the product repo root.

## Validation Guidance

- For TPF or step changes, run a build and inspect that the step appears in the GUI/status list.
- For framework changes, run framework tests and at least one consuming product build.
- For GUI changes, run the product app manually because WPF/operator behavior is not fully covered by compile checks.
- For packaging changes, inspect `output/<ft-name>_v<version>/` and the generated zip.

