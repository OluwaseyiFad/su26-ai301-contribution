# Contribution 1: Integration test for OTEL_DOTNET_AUTO_TRACES_ADDITIONAL_LEGACY_SOURCES

**Contribution Number:** 1  
**Student:** Oluwaseyi Fadahunsi
**Issue:** [GitHub issue link](https://github.com/open-telemetry/opentelemetry-dotnet-instrumentation/issues/2185)  
**Status:** Phase III — PR submitted and ready for review ([#5202](https://github.com/open-telemetry/opentelemetry-dotnet-instrumentation/pull/5202)); awaiting maintainer CI approval + review

---

## Why I Chose This Issue

I chose this issue because it is a focused way to contribute to OpenTelemetry without starting with a very large feature. It matches my interest in observability, testing, and understanding how .NET auto-instrumentation works in real projects.

I also want to improve my C#/.NET skills and learn how OpenTelemetry tests trace behavior end-to-end.

---

## Understanding the Issue

### Problem Description

The environment variable `OTEL_DOTNET_AUTO_TRACES_ADDITIONAL_LEGACY_SOURCES` lets users collect "legacy" activities — `Activity` objects created the old way (`new Activity("name")`) **without** an `ActivitySource`. The SDK ignores such activities unless their operation name is registered via `AddLegacySource`, which this variable drives. The feature works, but there is **no automated integration test** covering it. Issue #2185 asks for that missing test.

### Expected Behavior

There should be an integration test that runs an instrumented app emitting a legacy activity, sets `OTEL_DOTNET_AUTO_TRACES_ADDITIONAL_LEGACY_SOURCES`, and asserts the corresponding span is exported — protecting the feature from regressions, the same way `OTEL_DOTNET_AUTO_TRACES_ADDITIONAL_SOURCES` (the non-legacy variant) is already tested.

### Current Behavior

The feature works at runtime, but nothing guards it. A search of `test/IntegrationTests` for `ADDITIONAL_LEGACY_SOURCES` / `AddLegacySource` returns **no matches**, and no test application emits a raw `new Activity(...)`. A regression in legacy-source handling would ship undetected.

### Affected Components

- `src/OpenTelemetry.AutoInstrumentation/Configurations/TracerSettings.cs` — parses the env var into `AdditionalLegacySources`.
- `src/OpenTelemetry.AutoInstrumentation/Configurations/EnvironmentConfigurationTracerHelper.cs` — calls `builder.AddLegacySource(...)` for each entry.
- `test/IntegrationTests/` — where the new test belongs (`TestHelper`, `MockSpansCollector`, `EnvironmentHelper`).
- `test/test-applications/integrations/` + `OpenTelemetry.AutoInstrumentation.sln` — where a new test app is added and registered.

---

## Reproduction Process

### Environment Setup

**Prerequisites (per `docs/developing.md`):** Docker Desktop, .NET 8 SDK, .NET 9 SDK, Xcode Command Line Tools (macOS).

**Steps completed:**
1. Forked + cloned upstream; added remotes (`origin` = upstream, `fork` = my fork).
2. Created branch `add-legacy-sources-integration-test`.
3. Installed .NET SDKs into `~/.dotnet` (system `/usr/local/share/dotnet` needs sudo). Added `DOTNET_ROOT` + PATH to `~/.zshrc`.
4. `dotnet tool restore` → `dotnet nuke BuildTracer` succeeded; produced `bin/tracer-home` with the native `osx-arm64` profiler dylib.

**Challenge — nuke tool required .NET 10:** The repo pins `nuke.globaltool` 10.1.0, which ships only `net10.0` assets. With just .NET 8/9 installed, `dotnet tool restore` failed with `Settings file 'DotnetToolSettings.xml' was not found in the package`. A net10 tool needs the **.NET 10 SDK** to be resolved/run — not just the runtime. Installing the .NET 10 SDK (`dotnet-install.sh --channel 10.0`) fixed it. (So although the docs list 8 + 9, the build toolchain currently also needs .NET 10.)

### Steps to Reproduce

Because this is a *missing-test* issue, reproduction means demonstrating that (a) the feature works only when the variable is set, and (b) no test currently guards it. This was done **locally, outside the repo** (in `/tmp`) so the feature branch carries only submittable work.

1. Fork & clone `open-telemetry/opentelemetry-dotnet-instrumentation`; install prerequisites: .NET 8 & 9 SDKs — **plus .NET 10 SDK** (required by the pinned `nuke` 10.1.0 tool), Docker, and Xcode Command Line Tools.
2. Build the instrumentation: `dotnet tool restore && dotnet nuke BuildTracer` → produces `bin/tracer-home`.
3. In a scratch directory, create a minimal console app that emits one **legacy** activity:
   ```csharp
   using var activity = new Activity("ManualSpan");
   activity.Start();
   activity.Stop();
   ```
4. Run it instrumented with the console exporter — `OTEL_DOTNET_AUTO_HOME=bin/tracer-home`, `OTEL_TRACES_EXPORTER=console`, `OTEL_DOTNET_AUTO_LOG_DIRECTORY=/tmp/otel-logs` (the default `/var/log/opentelemetry` isn't writable on macOS) — **with** `OTEL_DOTNET_AUTO_TRACES_ADDITIONAL_LEGACY_SOURCES=ManualSpan`.
   → The `ManualSpan` span **is exported** (`Activity.DisplayName: ManualSpan`, `TraceFlags: Recorded`, empty instrumentation scope name).
5. Run it again **without** that variable.
   → The span is **not exported** (the app reports `Recorded=False`; no `Activity.*` output).
6. From the repo root, `grep -rn "ADDITIONAL_LEGACY_SOURCES\|AddLegacySource" test/IntegrationTests/` → **no matches**, confirming the test gap that #2185 asks to close.

### Reproduction Evidence

- **Commit showing reproduction:** https://github.com/OluwaseyiFad/opentelemetry-dotnet-instrumentation/commit/41564b9b96dfc0e00e433b1b2003e979a349afd5
- **Screenshots/logs:** Console-exporter output captured during reproduction —
  - **With** the env var: `Activity.DisplayName: ManualSpan`, `Activity.TraceFlags: Recorded`, `Instrumentation scope (ActivitySource): Name:` *(empty)*, and the app prints `recorded=True`.
  - **Without** the env var: only `[app] emitted legacy activity 'ManualSpan', recorded=False`; no span exported.
- **My findings:**
  - The variable directly gates collection: legacy activity → exported **iff** its name is listed.
  - **Legacy spans export under an *empty* instrumentation scope name** (not a named `ActivitySource`). This dictates the assertion: `collector.Expect("", span => span.Name == "ManualSpan")`.
  - The non-legacy `ADDITIONAL_SOURCES` variant *is* already integration-tested (e.g. `SmokeTests.cs`), giving a clear pattern to mirror.

---

## Solution Approach

### Analysis

This isn't a bug to fix — the root "cause" is a coverage gap. `OTEL_DOTNET_AUTO_TRACES_ADDITIONAL_LEGACY_SOURCES` flows from `TracerSettings.cs` into `EnvironmentConfigurationTracerHelper.cs`'s `AddLegacySource(...)` calls and works correctly, but nothing in `test/IntegrationTests` exercises that path. The fix is to add the missing end-to-end test (and the small test app it needs).

### Proposed Solution

Add a dedicated integration test that mirrors the existing `OTEL_DOTNET_AUTO_TRACES_ADDITIONAL_SOURCES` tests: a minimal instrumented test app that emits a legacy `Activity`, plus a test class that sets the legacy env var and asserts the span is collected via the mock collector (and, ideally, that it is *not* collected when the var is unset).

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** Provide automated integration coverage for `OTEL_DOTNET_AUTO_TRACES_ADDITIONAL_LEGACY_SOURCES`, which registers legacy `Activity` sources via `AddLegacySource`. Confirmed by reproduction that the feature works and is currently untested.

**Match:** The non-legacy variant is already covered. `SmokeTests.cs` (in `test/IntegrationTests`) shows the pattern: a `TestHelper`-derived class sets `OTEL_DOTNET_AUTO_TRACES_ADDITIONAL_SOURCES`, calls `RunTestApplication()`, and asserts spans with `MockSpansCollector.Expect(...)`. Test apps live under `test/test-applications/integrations/TestApplication.*` and are registered in `OpenTelemetry.AutoInstrumentation.sln`.

**Plan:**
1. Add test app `test/test-applications/integrations/TestApplication.TracesLegacySource/` (minimal `Program.cs` emitting `new Activity("ManualSpan")` + `.csproj`).
2. Register the new app project in `OpenTelemetry.AutoInstrumentation.sln`.
3. Add `LegacySourcesTests : TestHelper` in `test/IntegrationTests/` that sets `OTEL_DOTNET_AUTO_TRACES_ADDITIONAL_LEGACY_SOURCES`, runs the app, and asserts `collector.Expect("", span => span.Name == "ManualSpan")` (empty scope name — confirmed during reproduction).
4. (Stretch) Add a negative assertion: with the var unset, the legacy span is not collected.
5. Update `CHANGELOG.md`.

**Implement:** https://github.com/OluwaseyiFad/opentelemetry-dotnet-instrumentation/tree/add-legacy-sources-integration-test

**Review:** Self-check against `docs/CONTRIBUTING.md` — single concern, CHANGELOG updated, CLA signed, opened as draft PR; `dotnet nuke` build + format clean.

**Evaluate:** New test passes via `dotnet nuke ManagedTests` and fails if `AddLegacySource` wiring is removed (verifying it actually guards the feature).

---

## Testing Strategy

### Unit Tests

Not applicable — this is end-to-end behavior driven by an environment variable and the
auto-instrumentation profiler, which a unit test cannot exercise. Coverage is provided
through integration tests (matching how the non-legacy `ADDITIONAL_SOURCES` variant is tested).

### Integration Tests

New class `LegacySourcesTests : TestHelper` in `test/IntegrationTests/LegacySourcesTests.cs`,
backed by a new minimal test app `TestApplication.TracesLegacySource` that emits one legacy
activity via `new Activity("ManualSpan")`:

- [x] **Positive — `SubmitsLegacyActivityWhenSourceIsRegistered`:** with
  `OTEL_DOTNET_AUTO_TRACES_ADDITIONAL_LEGACY_SOURCES=ManualSpan`, the span is exported
  under the empty instrumentation scope name (`collector.Expect(string.Empty, span => span.Name == "ManualSpan")`).
- [x] **Negative — `DoesNotSubmitLegacyActivityWhenSourceIsNotRegistered`:** with the variable
  unset, no span is collected (`collector.AssertEmpty()`).

The positive/negative pairing is the regression guard: it proves collection is *gated* by the
variable, so a break in the `AddLegacySource` wiring (in `EnvironmentConfigurationTracerHelper.cs`)
would make the positive test fail.

### Manual Testing

[What you tested manually and results]

---

## Implementation Notes

### Week [X] Progress

[What you built this week, challenges faced, decisions made]

### Week [Y] Progress

[Continue documenting as you work]

### Code Changes

- **Files added:**
  - `test/test-applications/integrations/TestApplication.TracesLegacySource/Program.cs` — emits one legacy `new Activity("ManualSpan")`.
  - `test/test-applications/integrations/TestApplication.TracesLegacySource/TestApplication.TracesLegacySource.csproj` — minimal SDK project (TFMs/OutputType/Platforms inherited from `Integrations.props`).
  - `test/IntegrationTests/LegacySourcesTests.cs` — positive + negative integration tests.
- **Files modified:**
  - `OpenTelemetry.AutoInstrumentation.sln` — registered the new test app (project declaration, per-config platform rows, and `integrations` solution-folder nesting). Required because the nuke build discovers test apps from the solution.
- **Approach decisions:**
  - **Mirrored the existing `ADDITIONAL_SOURCES` pattern** (`SmokeTests` + `TestHelper` + `MockSpansCollector`) so the new test fits repo conventions.
  - **Empty scope-name assertion** (`Expect(string.Empty, …)`): legacy activities export with no `ActivitySource`, confirmed during Phase II reproduction.
  - **Added a negative test** so the pair proves the variable actually gates collection.

### Verification

- **Cross-TFM end-to-end** (real profiler + mock OTLP collector), 2/2 passing on each non-Windows target: `net8.0`, `net9.0`, `net10.0`. (`net462` is Windows-only — left to CI.)
- **Mutation test:** disabling the `AddLegacySource` wiring in `EnvironmentConfigurationTracerHelper.cs` (then rebuilding + swapping the managed assembly into `bin/tracer-home`) made the **positive** test fail while the **negative** test still passed — proving the test genuinely guards the feature. Reverted afterward; working tree left clean.

---

## Pull Request

**PR Link:** https://github.com/open-telemetry/opentelemetry-dotnet-instrumentation/pull/5202

**PR Description:** 
- **Why:** the legacy-sources env var is wired through `AddLegacySource` but has no integration coverage; a regression would ship undetected. `Fixes #2185`.
- **What:** new `TestApplication.TracesLegacySource` (emits one legacy `Activity`), new `LegacySourcesTests` (positive + negative), and solution registration.

**Maintainer Feedback:**
- [Date]: [Summary of feedback received]
- [Date]: [How you addressed it]

**Status:** Awaiting review

---

## Learnings & Reflections

### Technical Skills Gained

[What you learned technically]

### Challenges Overcome

[What was hard and how you solved it]

### What I'd Do Differently Next Time

[Reflection on your process]

---

## Resources Used

- [Link to helpful documentation]
- [Tutorial or Stack Overflow post that helped]
- [GitHub issues or discussions that helped]
