---
name: test-framework-dev
description: Develop Ampel functional-test framework projects and ft-amd-style test applications. Use when working on .NET 8 Windows FT apps, FTTest/FTTestStep code, TPF.json configuration, operator GUI integration, submodule wiring, database/login/label-printer/logger integrations, Makefile packaging, or test-framework architecture decisions.
---

# Test Framework Dev

## Operating Model

Use the repository as a multi-repo FT workspace: a product app owns product-specific test flow and references shared submodules for framework execution, GUI, logging, database, login, and label printing.

Before editing, inspect the current local checkout because product repos pin submodule commits and framework APIs may drift. Prefer the code in the checkout over memory.

## First Pass

1. Identify whether the task touches the product app, the framework, or a shared submodule.
2. Inspect the relevant `.csproj`, `.sln`, `.gitmodules`, `TPF.json`, and `Makefile`.
3. Read only the reference file needed for the task:
   - `references/architecture.md` for repository shape, execution flow, and ownership boundaries.
   - `references/tpf-and-test-steps.md` for adding or changing FT test steps and TPF modules.
   - `references/project-template-ft-amd.md` for the observed ft-amd app pattern.
   - `references/submodules.md` for shared library roles and likely edit locations.
   - `references/build-package-release.md` for build, package, version, and validation commands.
4. Keep app-specific behavior in the product app unless the change is reusable across products.
5. Treat `appsettings.json`, API keys, tokens, device IDs, database hosts, and serial numbers as sensitive. Do not copy real values into docs, tests, or examples.

## Development Rules

- Use `FTTest<TTest>` for a product test aggregate and `FTTestStep<TTest>` for individual steps.
- Keep `VisualizerName()` exactly aligned with the `TPF.json` step key when `IncludedInsideTpf()` returns `true`.
- Add each new step to the product step-order enum and return that enum value from `GetOrderTestStep()`.
- Use `SharedResources` for product state, user events, clients, and cross-step data.
- Use framework `Assert`/`FTAssert` methods for test expectations so failures are collected in `TestResult`.
- Keep hardware, serial, cloud, database, and GUI side effects behind existing product abstractions where possible.
- Validate with the narrowest reliable command first, then build/package only when the change affects integration or release output.

## Common Tasks

- **Add a test step**: read `references/tpf-and-test-steps.md`, then mirror the ft-amd pattern in `test-app-amd/tests/*`.
- **Change framework behavior**: read `references/architecture.md`, then inspect `ft-test-framework/TestFramework/Api` and `ft-test-framework/TestFramework/Core`.
- **Change UI/operator flow**: read `references/project-template-ft-amd.md` and `references/submodules.md`, then inspect `TAInterface`, `TaEvents`, `GuiInterface`, and `ft-gui`.
- **Change package or release output**: read `references/build-package-release.md`, then inspect the root `Makefile` and main app `.csproj`.
- **Change auth, logging, database, or labels**: read `references/submodules.md`, then inspect the matching submodule before modifying the product app.

