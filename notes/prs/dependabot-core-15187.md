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

## 육하원칙으로 보기

| 질문 | 답 |
| --- | --- |
| 누가 | DamianEdwards가 PR을 열었고, JamieMagee, jjonescz, brettfo가 리뷰했다. |
| 언제 | PR은 2026-05-30에 열렸고, 주요 리뷰는 2026-06-07, 2026-06-08, 2026-06-18에 진행됐다. |
| 어디서 | `dependabot-core`의 NuGet updater 영역, 특히 `NuGetUpdater.Core/Discover`, `Updater/FileWriters`, `DependencySolver`, `ExperimentsManager`에서 진행됐다. |
| 무엇을 | C# file-based app, 즉 `.csproj` 없이 `dotnet run file.cs`로 실행되는 단일 C# 파일 안의 `#:package` NuGet dependency를 Dependabot이 발견하고 업데이트하게 하려 했다. |
| 왜 | 관련 이슈 #12057에서 .NET의 새 file-based app 문법이 제안됐고, 그 문법 안에 NuGet package metadata가 들어가므로 Dependabot도 dependency로 인식해야 했기 때문이다. |
| 어떻게 | 새 discovery 코드로 `.cs` 파일을 찾고, directive block에서 `#:package`를 파싱해 `ProjectDiscoveryResult`로 만들고, 새 file writer가 해당 directive의 version을 바꾸도록 했다. |

## 5W1H In English

| Question | Answer |
| --- | --- |
| Who | DamianEdwards opened the PR; JamieMagee, jjonescz, and brettfo reviewed it. |
| When | The PR was opened on 2026-05-30. The important review rounds happened on 2026-06-07, 2026-06-08, and 2026-06-18. |
| Where | The change lives in the NuGet updater path: `Discover`, `Updater/FileWriters`, `DependencySolver`, and `ExperimentsManager`. |
| What | It adds support for NuGet dependencies declared inside C# file-based apps, where a single `.cs` file can contain `#:package` directives. |
| Why | Issue #12057 requested support for the new .NET `dotnet run file.cs` metadata model, where package references can live directly in the source file. |
| How | The PR adds discovery for standalone `.cs` files, parses the directive block, converts package directives into Dependabot dependency records, and writes updated versions back into the source file. |

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

## 기존 코드에서 뭐가 비어 있었나

기존 NuGet updater의 기본 흐름은 project/workspace 중심이다.

```text
DiscoveryWorker
  -> find project files such as .csproj
  -> discover dependencies from project/lock/config files
  -> update project files through existing file writers
```

그런데 C# file-based app은 `.csproj`가 없다. dependency가 `.cs` 파일 상단의 directive에 들어간다.

```csharp
#:package System.CommandLine@2.0.0-*
```

그래서 기존 pipeline에는 이런 파일을 dependency source로 보는 길이 없었다. 이 PR의 출발점은 "버그 난 줄 고치는 것"보다 "새로운 입력 형식을 기존 NuGet updater architecture에 연결하는 것"에 가깝다.

## What Was Missing In The Existing Code

The existing NuGet updater path is project/workspace oriented.

```text
DiscoveryWorker
  -> find project files such as .csproj
  -> discover dependencies from project/lock/config files
  -> update project files through existing file writers
```

C# file-based apps do not have a `.csproj`. Their dependencies can live directly in a `.cs` source file as directives.

```csharp
#:package System.CommandLine@2.0.0-*
```

So the old pipeline had no dependency-source path for this format. The PR is less "fix a broken line" and more "connect a new ecosystem input shape to Dependabot's NuGet architecture."

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

## 코드 진행 흐름

### 1. DiscoveryWorker에 새 입력 형식 연결

기존 `DiscoveryWorker`는 project file을 찾는다. PR은 여기에 file-based app discovery를 붙인다.

```text
RunForDirectoryAsync
  -> find project files
  -> if experiment allows it, run CSharpFileBasedAppDiscovery
  -> concat normal project results + file-based app results
```

중요한 설계 포인트:

- `.csproj`가 없어도 결과가 나올 수 있어야 한다.
- 하지만 일반 project 아래의 `.cs` 파일은 file-based app으로 오인하면 안 된다.
- 그래서 `IsInProjectCone`로 project directory 아래 `.cs` 파일을 제외한다.

### 2. CSharpFileBasedAppDiscovery가 `.cs` 파일을 dependency source로 해석

새 discovery 코드는 workspace 안의 `.cs` 파일을 순회한다.

```text
Enumerate *.cs
  -> exclude files under .csproj directories
  -> determine default target framework
  -> read directive block
  -> parse #:package directives
  -> emit ProjectDiscoveryResult
```

여기서 PR의 핵심 코드 판단은 `GetPackageDependencies`다. 이 함수는 파일을 위에서부터 읽으며 directive block만 본다.

```text
for each line:
  if line is not blank/comment/shebang/#: directive:
    stop scanning
  if line matches #:package:
    create Dependency
```

이 흐름은 1차 리뷰 뒤에 보강된 것으로 보인다. 리뷰어가 "파일 어디에서든 `#:package`를 찾으면 C# string literal 안의 가짜 dependency를 잡는다"고 지적했기 때문이다.

### 3. Target framework를 구해야 dependency model이 완성됨

file-based app에는 명시적인 project file이 없기 때문에 기본 target framework가 문제 된다. PR은 임시 `.cs` 파일을 만들고 `dotnet build ... -getProperty:TargetFramework`로 SDK가 보는 값을 가져오려 한다.

실패하면 SDK major version 또는 현재 runtime major version으로 fallback한다.

리뷰 포인트:

- target framework는 SDK 버전에 따라 바뀔 수 있다.
- 그래서 hard-code하면 안 된다.
- 가능하면 실제 SDK/MSBuild에게 물어봐야 한다.

### 4. FileWriter가 source file의 directive만 수정

새 `CSharpFileBasedAppFileWriter`는 `.cs` 파일의 directive block에서 versioned `#:package`만 바꾼다.

```text
original dependency versions
required dependency versions
  -> scan directive block
  -> find matching package name
  -> update exact version or version range
  -> preserve line endings and encoding
```

문제는 writer도 discovery와 같은 parsing semantics를 가져야 한다는 점이다. discovery는 가짜 dependency를 무시했는데 writer가 다른 regex로 더 넓게 잡으면, 발견하지 말아야 할 텍스트를 수정할 수 있다.

### 5. DependencySolver 우회가 리뷰에 걸림

PR의 현재 patch에는 `.cs` file-based app이면 solver가 그냥 `desiredDependencies`를 반환하는 흐름이 있다.

```text
if project extension is .cs:
  return desiredDependencies
```

brettfo의 리뷰는 이 부분이 핵심이다. file-based app이어도 dependency conflict는 생긴다. 어떤 dependency는 다른 package와 버전이 묶여 있고, solver가 transitive dependency를 top-level로 끌어올려야 할 수도 있다.

즉, 입력 형식이 `.cs`일 뿐 dependency graph 문제는 사라지지 않는다.

### 6. Experiment flag 정책도 architecture다

PR은 `nuget_update_file_based_apps` experiment와 `update-file-based-apps` job option을 추가했다. 그런데 리뷰에서 experiment 값은 기본적으로 false여야 한다고 지적했다.

학습 포인트:

- feature flag는 단순 if문이 아니다.
- rollout safety, default behavior, opt-in/opt-out 정책을 담는다.
- Dependabot 같은 자동 PR 생성기는 새 동작을 조심스럽게 켜야 한다.

## Code Flow In English

### 1. Wire the new input format into DiscoveryWorker

The PR extends `DiscoveryWorker` so normal project discovery and file-based app discovery can both produce `ProjectDiscoveryResult` values.

```text
RunForDirectoryAsync
  -> find project files
  -> if the experiment allows it, run CSharpFileBasedAppDiscovery
  -> concatenate normal project results and file-based app results
```

Important design constraints:

- A dependency result can exist even when there is no `.csproj`.
- A normal `.cs` file under a real project must not be misclassified as a file-based app.
- The PR excludes `.cs` files inside project directories with `IsInProjectCone`.

### 2. Treat standalone `.cs` files as dependency sources

`CSharpFileBasedAppDiscovery` walks `.cs` files and tries to interpret only the top directive block.

```text
Enumerate *.cs
  -> exclude files under .csproj directories
  -> determine default target framework
  -> read directive block
  -> parse #:package directives
  -> emit ProjectDiscoveryResult
```

The critical function is `GetPackageDependencies`. It scans from the top of the file and stops once it reaches actual C# code.

```text
for each line:
  if line is not blank/comment/shebang/#: directive:
    stop scanning
  if line matches #:package:
    create Dependency
```

This was strengthened after review feedback: scanning the whole file would treat directive-looking text inside string literals as real dependencies.

### 3. Infer target framework from the SDK

Because there is no project file, the PR needs a target framework. It creates a temporary `.cs` file and asks `dotnet build ... -getProperty:TargetFramework` what the SDK would use.

If that fails, it falls back to SDK major version or current runtime major version.

The review lesson:

- target framework defaults are SDK semantics
- Dependabot should not casually hard-code them
- the safest source of truth is the actual SDK/MSBuild behavior

### 4. Update only package directives in the source file

`CSharpFileBasedAppFileWriter` updates versioned `#:package` directives in the directive block.

```text
original dependency versions
required dependency versions
  -> scan directive block
  -> match package name
  -> update exact version or version range
  -> preserve line endings and encoding
```

The writer must use the same semantics as discovery. If discovery ignores a fake dependency but the writer's regex matches it, Dependabot could modify normal C# code.

### 5. Solver bypass is the biggest architecture objection

The current patch short-circuits the solver for `.cs` file-based apps.

```text
if project extension is .cs:
  return desiredDependencies
```

The requested-changes review says this is unsafe. File-based apps can still have dependency conflicts, locked version relationships, and transitive dependencies that need to be promoted to top-level package directives.

The input format changed; the dependency graph problem did not.

### 6. Experiment flags are part of the architecture

The PR adds `nuget_update_file_based_apps` and `update-file-based-apps`. Review pushed back because Dependabot experiments are expected to default to false.

The lesson:

- a feature flag is not just a boolean
- it encodes rollout safety and product policy
- an automated updater should introduce new behavior carefully

## Review Conversation: What Actually Got Challenged

### 1. Regex Is Not Enough For C# Parsing

Initial direction appears to scan for `#:package`-looking lines. Reviewers pushed back because a directive-looking string inside normal C# code can become a fake dependency.

The deeper lesson: in Dependabot, false positives are expensive. A phantom dependency can make the bot open a wrong update PR, which breaks user trust.

Problem shape:

```csharp
var text = """
#:package Foo@1.2.3
""";
```

If Dependabot treats that as a real package directive, it invents a dependency that the SDK would not use.

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

This is the most important architecture critique in the PR. The author tried to treat file-based apps as simpler than `.csproj` projects, but the reviewer pulled the design back toward the existing NuGet solver model.

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

## 문제 코드와 리뷰 반응 요약

| 코드/설계 | 왜 문제였나 | 리뷰 방향 |
| --- | --- | --- |
| `#:package` regex parsing | C# string literal/comment 안의 가짜 directive를 dependency로 오인할 수 있다. | Roslyn 같은 C# parser 또는 SDK와 같은 semantics를 쓰라는 방향. |
| 파일 전체 scanning | SDK는 파일 상단 directive block만 metadata로 본다. | 실제 C# code를 만나면 scanning 중단. |
| target framework hard-code/fallback | SDK 버전에 따라 기본 target framework가 바뀐다. | 가능하면 `dotnet build -getProperty:TargetFramework`로 SDK에게 물어보기. |
| `.cs`면 solver bypass | file-based app도 dependency conflict가 생긴다. | 기존 MSBuild/Sdk discovery/solver infrastructure 재사용. |
| 자체 BOM/newline 처리 | 이미 `ModifiedFilesTracker`가 파일 포맷 추적을 담당한다. | 새 writer가 기존 updater runner infrastructure에 붙어야 함. |
| experiment default true | Dependabot experiment는 안전하게 default false여야 한다. | 기존 experiment 정책 유지. |

## Problem Code And Review Reaction

| Code/design | Why it was risky | Review direction |
| --- | --- | --- |
| `#:package` regex parsing | It can treat fake directives inside C# strings/comments as real dependencies. | Use Roslyn or otherwise match SDK parsing semantics. |
| Whole-file scanning | The SDK only treats the top directive block as metadata. | Stop scanning when actual C# code starts. |
| Hard-coded/fallback target framework | The default target framework depends on the SDK version. | Ask the SDK/MSBuild, for example through `dotnet build -getProperty:TargetFramework`. |
| Solver bypass for `.cs` files | File-based apps can still have dependency conflicts. | Reuse existing MSBuild/Sdk discovery and solver infrastructure. |
| Custom BOM/newline handling | Existing updater infrastructure already tracks formatting details. | Integrate with `ModifiedFilesTracker`. |
| Experiment defaulting to true | Dependabot experiments are expected to default to false. | Keep the established experiment rollout policy. |

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
