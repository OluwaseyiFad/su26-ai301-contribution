# Contribution 1: Integration test for OTEL_DOTNET_AUTO_TRACES_ADDITIONAL_LEGACY_SOURCES

**Contribution Number:** 1  
**Student:** Oluwaseyi Fadahunsi
**Issue:** [GitHub issue link](https://github.com/open-telemetry/opentelemetry-dotnet-instrumentation/issues/2185)  
**Status:** Phase I

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

1. [Step 1]
2. [Step 2]
3. [Observed result]

### Reproduction Evidence

- **Commit showing reproduction:** [Link to commit in your fork]
- **Screenshots/logs:** [If applicable]
- **My findings:** [What you discovered during reproduction]

---

## Solution Approach

### Analysis

This isn't a bug to fix — the root "cause" is a coverage gap. `OTEL_DOTNET_AUTO_TRACES_ADDITIONAL_LEGACY_SOURCES` flows from `TracerSettings.cs` into `EnvironmentConfigurationTracerHelper.cs`'s `AddLegacySource(...)` calls and works correctly, but nothing in `test/IntegrationTests` exercises that path. The fix is to add the missing end-to-end test (and the small test app it needs).

### Proposed Solution

Add a dedicated integration test that mirrors the existing `OTEL_DOTNET_AUTO_TRACES_ADDITIONAL_SOURCES` tests: a minimal instrumented test app that emits a legacy `Activity`, plus a test class that sets the legacy env var and asserts the span is collected via the mock collector (and, ideally, that it is *not* collected when the var is unset).

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** [Restate the problem]

**Match:** [What similar patterns/solutions exist in the codebase?]

**Plan:** [Step-by-step implementation plan]
1. [Modify file X to do Y]
2. [Add function Z]
3. [Update tests]

**Implement:** [Link to your branch/commits as you work]

**Review:** [Self-review checklist - does it follow the project's contribution guidelines?]

**Evaluate:** [How will you verify it works?]

---

## Testing Strategy

### Unit Tests

- [ ] Test case 1: [Description]
- [ ] Test case 2: [Description]
- [ ] Test case 3: [Description]

### Integration Tests

- [ ] Integration scenario 1
- [ ] Integration scenario 2

### Manual Testing

[What you tested manually and results]

---

## Implementation Notes

### Week [X] Progress

[What you built this week, challenges faced, decisions made]

### Week [Y] Progress

[Continue documenting as you work]

### Code Changes

- **Files modified:** [List]
- **Key commits:** [Links to important commits]
- **Approach decisions:** [Why you chose certain approaches]

---

## Pull Request

**PR Link:** [GitHub PR URL when submitted]

**PR Description:** [Draft or final PR description - much of the content above can be adapted]

**Maintainer Feedback:**
- [Date]: [Summary of feedback received]
- [Date]: [How you addressed it]

**Status:** [Awaiting review / Iterating / Approved / Merged]

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
