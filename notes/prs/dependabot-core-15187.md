# dependabot-core PR #15187 - C# file-based apps

Source: https://github.com/dependabot/dependabot-core/pull/15187

Status checked: 2026-07-16 KST

## Snapshot

- PR: `dependabot/dependabot-core#15187`
- Title: Add NuGet support for C# file-based apps
- Author: DamianEdwards
- State: open
- Merge status: not merged
- Review status: changes requested
- Size: 16 files, 9 commits, about +1374 / -23
- Related issue: https://github.com/dependabot/dependabot-core/issues/12057

This is not a "passed PR" yet. It is still more valuable as a study object because the review shows exactly where a clean-looking dependency update feature gets hard.

## What The PR Tries To Add

.NET is adding support for running a single C# file directly with project metadata written as directives near the top of the file.

Example shape:

```csharp
#:sdk Microsoft.NET.Sdk.Web
#:property TargetFramework net11.0
#:package System.CommandLine 2.0.0-*
```

Dependabot needs to:

- discover standalone `.cs` file-based apps
- extract NuGet package directives
- update versioned `#:package` directives
- avoid touching ordinary C# files inside normal project/workspace structure
- preserve comments, formatting, BOM, and newline style
- allow the behavior to be controlled by an experiment/job option

## Files Worth Reading First

| File | Why It Matters |
| --- | --- |
| `NuGetUpdater.Core/Discover/CSharpFileBasedAppDiscovery.cs` | Discovers file-based apps and parses package metadata. This is where parsing strategy becomes the fight. |
| `NuGetUpdater.Core/Updater/FileWriters/CSharpFileBasedAppFileWriter.cs` | Rewrites package directives while preserving the user's file. |
| `NuGetUpdater.Core/DependencySolver/MSBuildDependencySolver.cs` | Decides whether file-based apps still need dependency conflict solving. |
| `NuGetUpdater.Core/Discover/DiscoveryWorker.cs` | Integrates file-based app discovery into the broader NuGet discovery pipeline. |
| `NuGetUpdater.Core/ExperimentsManager.cs` | Shows how feature flags/experiments control rollout. |
| `NuGetUpdater.Core/Utilities/MSBuildHelper.cs` | Existing helper that reviewers wanted reused instead of inventing a parallel path. |
| `NuGetUpdater.Core.Test/...FileBasedApp...` | Tests reveal the real edge cases: strings, comments, lockfiles, opt-outs, formatting. |

## Review Conversation: What Actually Got Challenged

### 1. Regex Is Not Enough For C# Parsing

Initial direction appears to scan for `#:package`-looking lines. Reviewers pushed back because a directive-looking string inside normal C# code can become a fake dependency.

The deeper lesson: in Dependabot, false positives are expensive. A phantom dependency can make the bot open a wrong update PR, which breaks user trust.

Study question:

- When is a regex acceptable?
- When does the language grammar force a real parser?
- What does "parse like the SDK" mean in practice?

### 2. Directive Scope Matters

The .NET SDK only treats directives as metadata before real C# code begins. A package-looking line after the directive block should not count.

The deeper lesson: Dependabot cannot invent a looser interpretation than the ecosystem's own toolchain. It has to match the package manager/runtime semantics closely.

Study question:

- Where does the directive block end?
- What counts as blank, comment, shebang, directive, or actual C# code?
- How do tests prove that a string literal is not a dependency?

### 3. Default Target Framework Cannot Be Guessed Casually

Review discussion questioned hard-coding a default target framework because it depends on the .NET SDK being used.

The deeper lesson: environment-derived defaults are part of the build semantics. If Dependabot guesses wrong, dependency resolution can differ from the user's real build.

Study question:

- Should Dependabot ask the SDK for the target framework?
- Is `dotnet build -getProperty:TargetFramework` the right source of truth?
- What happens when the SDK changes its default?

### 4. SDK Directives Are A Boundary Decision

Reviewers asked whether `#:sdk` directives should be handled too. The author noted Dependabot does not currently handle SDKs for traditional projects in the same way.

The deeper lesson: feature scope is not just "can we parse it?" It is "does this belong to Dependabot's dependency model?"

Study question:

- Is an SDK directive a dependency?
- If NuGet itself does not manage it, should Dependabot?
- Is this PR about packages only, or file-based project metadata generally?

### 5. Dependency Solving Still Applies

One review point challenged bypassing dependency conflict solving for file-based apps. File-based apps can still have locked or coupled dependency versions.

The deeper lesson: changing the input format does not remove the dependency graph problem. The same constraint solving pressure comes back through a new doorway.

Study question:

- Can a transitive dependency need promotion to top-level?
- What does `requiredPackageVersions` mean after solving?
- How should the file writer react when the solver returns more than the original top-level set?

### 6. Reuse Existing Infrastructure

Reviewers pointed to existing helpers such as temp project creation, SDK project discovery, and modified-file tracking.

The deeper lesson: mature codebases punish parallel implementations. If newline/BOM tracking already lives in one subsystem, a new writer should join that subsystem.

Study question:

- What infrastructure already exists?
- What invariants does it preserve?
- What breaks if the new path reimplements only 80 percent of it?

### 7. Experiment Flags Have Policy

The review objected to experiment values defaulting in a new way. The project expects experiment values to default to false.

The deeper lesson: rollout mechanics are architecture. A feature flag is not a random boolean; it encodes deployment and safety policy.

Study question:

- What is the default behavior for a new updater feature?
- Is this opt-in, opt-out, or experiment-gated?
- How do job options map into experiment flags?

## Architecture Map

```text
job config / experiment flag
  -> DiscoveryWorker
  -> CSharpFileBasedAppDiscovery
  -> MSBuild / SDK-derived semantics
  -> dependency model
  -> MSBuildDependencySolver
  -> CSharpFileBasedAppFileWriter
  -> ModifiedFilesTracker / updater runner
```

The key architecture pressure is that file-based apps look like plain `.cs` files, but Dependabot must treat them like miniature projects only when the SDK would do the same.

## Why This Is A Good Dependabot Study PR

This PR touches the central Dependabot pattern:

```text
find files
  -> parse ecosystem-specific dependency declarations
  -> resolve versions using ecosystem semantics
  -> write minimal, trustworthy file changes
  -> prove behavior with targeted tests
```

The hard part is not C# syntax alone. The hard part is preserving the user's mental model: Dependabot should update what the real .NET tooling would treat as a package dependency, and nothing else.

## How To Read The Conversation

Read in this order:

1. PR summary and related issue #12057.
2. First review comments about false dependencies from strings and directive scope.
3. Follow-up commits named "Address NuGet file app review feedback".
4. Discussion about target framework and SDK-derived defaults.
5. Later requested-changes review about dependency solving and existing infrastructure.
6. Files changed, especially discovery, writer, solver, and tests.

## Open Questions To Track

- Will the final implementation use Roslyn instead of regex?
- Will it ask the .NET SDK/MSBuild for target framework and dependency metadata?
- Will file-based apps go through the same dependency solver as project files?
- Will the file writer preserve BOM/newlines through `ModifiedFilesTracker`?
- Will experiment behavior stay consistent with Dependabot's default-false policy?

## Takeaway

This PR shows Dependabot's real difficulty: every ecosystem feature is not just parsing text. It is reproducing another toolchain's semantics closely enough that automated PRs remain boring, correct, and trusted.
