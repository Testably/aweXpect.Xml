# Development workflows

All commands run from the repository root. `build.ps1` (Windows), `build.sh` (Linux and macOS) and `build.cmd` start the build pipeline in `Pipeline/`; plain `dotnet` commands work for the inner loop.

## Build and test

- `dotnet build aweXpect.Xml.slnx` builds the library, tests, AOT smoke tests and benchmarks. The pipeline project is excluded from the solution build. Building the library also packs it (`GeneratePackageOnBuild`).
- `dotnet test Tests/aweXpect.Xml.Tests -f net10.0` runs the unit tests (xunit v2) for one framework. Use `-f net8.0` or, on Windows, `-f net48` for the others.
- Filter by the nested test classes, e.g. `--filter "FullyQualifiedName~ThatXmlString+IsValidXml"` or `--filter "FullyQualifiedName~XmlOptions+IgnoringMember"`.
- Add `--no-build` after a solution build to skip the rebuild.
- Failure messages of `System.Xml` are localized on .NET Framework. Tests that assert such a message use `[UseEnglishMessages]` from `Tests/aweXpect.Xml.Tests/Helpers`.

## Public API

- `dotnet test Tests/aweXpect.Xml.Api.Tests` (NUnit, `net8.0`) reads the built `aweXpect.Xml.dll` of every target framework listed in `Source/Directory.Build.props` and compares its public API with `ApiApprovalTests.aweXpect%2EXml.<framework>.awexpect-snapshot.txt` via `aweXpect.Snapshots.PublicApi`. The project builds all target frameworks of the library itself.
- A difference fails the test and writes a `*.awexpect-snapshot.pending.txt` next to the snapshot. Accept an intended change by renaming the pending file, running the `[Explicit]` test `AcceptApiChanges`, or running the tests with `AWEXPECT_ACCEPT_SNAPSHOTS=all`. Commit only the updated `*.awexpect-snapshot.txt` files.
- The tests pick the `Debug` or `Release` output matching their own configuration, so build and test in the same configuration.

## AOT smoke tests

- `Tests/Aot/aweXpect.Xml.Aot` is a console application (`net10.0`, `PublishAot`) whose checks in `Checks.cs` exercise every expectation and report through the exit code. Trimmer and AOT compiler warnings are errors.
- `dotnet run --project Tests/Aot/aweXpect.Xml.Aot` runs the checks under the JIT; `./build.ps1 AotSmokeTests` runs them under the JIT and again after a Native AOT publish for the current platform (needs the native toolchain, e.g. the Visual Studio C++ build tools on Windows).

## Benchmarks

- `Benchmarks/aweXpect.Xml.Benchmarks` uses BenchmarkDotNet. Add benchmarks to the partial class `HappyCaseBenchmarks` with a method name ending in `_aweXpect`: the pipeline reads its GitHub report and highlights these rows in the pull request comment.
- Run locally with `dotnet run -c Release --project Benchmarks/aweXpect.Xml.Benchmarks -- --filter *`, or through the pipeline with `./build.ps1 Benchmarks`.

## Mutation tests

- `./build.ps1 MutationTests` installs `dotnet-stryker` into `TestResults/` and mutates `Source/aweXpect.Xml` against `aweXpect.Xml.Tests` on `net10.0`. Outside `main` only code changed since `main` is mutated. Reports land in `Artifacts/Stryker`. This is slow; leave it to CI unless asked.

## Documentation

- `README.md` is the package readme and the start page of the documentation site. `Pack` rewrites its badges for the release into `Artifacts/README.md`.
- `Docs/pages/*.md` are published to [docs.testably.org](https://docs.testably.org/aweXpect/extensions/aweXpect.Xml/): after a successful `Build` workflow on `main`, `notify-docs-site.yml` triggers a rebuild of `Testably/Testably.Site`. `00-index.md` embeds the README through `{README}`. Update the pages and the README overview table when an expectation is added or changed.
- Code fences in the docs are not compiled, so check examples against a test.
- `Docs/agents/` holds guides for authors and agents and is not published.

## Build pipeline

The pipeline is a Fallout build (`Pipeline/Build.csproj`, configured in `.fallout/parameters.json`). `./build.ps1 --help` lists the targets. The configuration defaults to `Debug` locally and `Release` on a server (`--configuration`). Running without a target runs `ApiChecks`, `Benchmarks` and `CodeAnalysis`.

| Target                                                  | Purpose                                                                                                        |
|---------------------------------------------------------|----------------------------------------------------------------------------------------------------------------|
| `Compile`                                               | Cleans `Artifacts/` and `TestResults/`, restores and builds the solution with the GitVersion version           |
| `UnitTests`                                             | Runs `aweXpect.Xml.Tests` for `net8.0` and `net10.0` with coverage, and for `net48` with the xunit console on Windows |
| `CodeCoverage`                                          | `UnitTests` plus an OpenCover report in `TestResults/reports`                                                   |
| `ApiChecks`                                             | Runs `aweXpect.Xml.Api.Tests`                                                                                  |
| `AotSmokeTests`                                         | Runs the AOT smoke tests under the JIT and published with Native AOT                                            |
| `Benchmarks`                                            | Runs all benchmarks and logs the report to `Artifacts/Benchmarks`                                               |
| `MutationTests`                                         | Runs Stryker and prepares the pull request comment                                                             |
| `CodeAnalysis`                                          | SonarCloud analysis around `CodeCoverage`; needs `SONAR_TOKEN`                                                  |
| `Pack`                                                  | Moves the `.nupkg` and `.snupkg` into `Artifacts/Packages`                                                      |
| `BenchmarkComment`, `MutationTestDashboard`             | CI only: download the artifacts of a CI run and comment on the pull request or upload to the Stryker dashboard |

## CI workflows

- `ci.yml` (pull requests to `main`): unit tests with coverage on Ubuntu, Windows and macOS, AOT smoke tests on all three, API tests, mutation tests, benchmarks, SonarCloud (not for forks or Dependabot) and published test results.
- `ci-analysis.yml` (after `CI`): posts the benchmark comment and uploads the mutation report, which need repository secrets.
- `build.yml` (pushes to `main` and `v*` tags): the same checks plus `Pack`. A tag `vX.Y.Z` pushes the packages to nuget.org and creates a GitHub release; a pre-release tag like `vX.Y.Z-beta.1` only pushes the packages.
- `notify-docs-site.yml`: rebuilds the documentation site after a successful `Build` on `main`.
- Dependabot updates NuGet packages and actions weekly with `chore:` commits; Mergify squash-merges its pull requests once the unit and API tests pass.
