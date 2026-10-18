# aweXpect.Xml Agent Guide

aweXpect.Xml is an extension of [aweXpect](https://github.com/Testably/aweXpect) with expectations for XML strings, `XElement` and `XDocument`. It ships as a single NuGet package from `Source/aweXpect.Xml`.

## Setup

- Use the SDK pinned in `global.json` (10.0.300, latest patch). Running the tests needs the .NET 8 and .NET 10 runtimes; the `net48` tests only run on Windows.
- Package versions are managed centrally in `Directory.Packages.props`. Shared settings live in `Source/Directory.Build.props` (library) and `Tests/Directory.Build.props` (tests); the root `Directory.Build.props` signs every assembly with `strongname.snk`.
- The library treats warnings as errors (except missing XML docs, CS1591). Fix a warning instead of suppressing it, and ask before adding a `#pragma`, `NoWarn` entry or `.editorconfig` severity change.

## Design goals

Correctness is a must. The other goals are strong defaults and win in this order when they conflict: stable and compatible, AOT and trimming friendly, a consistent and pleasant API, a fast happy path, extensible, errors at compile time, documented. [Docs/agents/design-goals.md](Docs/agents/design-goals.md) explains each goal, the code rules, the checks before a commit and the goals of this repository.

## Project constraints

- The library targets `netstandard2.0;net8.0;net10.0`; the tests target `net8.0;net10.0;net48`. Test code must compile for .NET Framework 4.8: no records, no list patterns, no APIs missing there.
- The library depends only on `aweXpect.Core`. Its version in `Directory.Packages.props` is the minimum that consumers get, so raise it only when a needed Core feature requires it. Tests and benchmarks use the full `aweXpect` package.
- Every public expectation on `IThat<T>` is an extension method in a `ThatXmlString`, `ThatXElement` or `ThatXDocument` partial class (namespace `aweXpect`), one file per expectation, and carries `[GuaranteesNotNull]`. Options for string equality, like `AsXml()`, go into `XmlExtensions`.
- XML is parsed only through `Helpers/SecureXml`: DTD processing is prohibited and no external resource is resolved. Never parse untrusted XML with `XDocument.Parse` or default `XmlReaderSettings`.
- Keep the library trimmable and Native AOT compatible (`IsTrimmable` and `IsAotCompatible` from `net8.0` on). Do not add unguarded reflection.
- Public API changes are deliberate: every addition needs XML docs and an updated API snapshot. Docs, XML docs and commit messages use no em-dashes.

## Validation

- Run the tests for your change before pushing, e.g. `dotnet test Tests/aweXpect.Xml.Tests -f net10.0 --filter "FullyQualifiedName~ThatXElement+HasAttribute"`. Run `-f net48` too when you use a newer language feature.
- Assert failures with `Throws<XunitException>().WithMessage(...)` on the full message, never a bare `Throws`. Explain non-obvious cases with `.Because(...)` instead of a comment.
- A public API change fails `Tests/aweXpect.Xml.Api.Tests`. Review the `*.awexpect-snapshot.pending.txt` files it writes, then accept them (rename, run the explicit `AcceptApiChanges` test, or set `AWEXPECT_ACCEPT_SNAPSHOTS=all`) and commit the `*.awexpect-snapshot.txt` files, never the pending ones.
- When an expectation changes, extend the AOT smoke checks in `Tests/Aot/aweXpect.Xml.Aot/Checks.cs` and run `./build.ps1 AotSmokeTests`.
- Never run `./build.ps1` without a target: the defaults include benchmarks and the SonarCloud analysis.

## Git workflow

- Branch from `main` and open a pull request; pull requests are squash-merged once the CI workflow passes. Do not commit `.claude/` or other local notes.
- Commit subjects follow conventional commits (`feat:`, `fix:`, `test:`, `docs:`, `ci:`, `chore:`) and double as the PR title. The body is the PR description: explain why, keep each paragraph on a single line, and add no co-author or attribution trailers.
- Many files carry a UTF-8 BOM and the working tree uses CRLF. Prefer targeted edits over rewriting files from PowerShell or `sed`, and check `git diff` for added or removed BOMs (U+FEFF) and broken raw string literals before committing.

## References

- [Docs/agents/design-goals.md](Docs/agents/design-goals.md): the goals and code rules for every change, the checks before a commit and the goals of this repository.
- [Docs/agents/workflows.md](Docs/agents/workflows.md): build, test, benchmark, mutation and documentation commands, the `Pipeline/` targets and the CI workflows.
- [Docs/agents/project-structure.md](Docs/agents/project-structure.md): architecture and source locations.
- [CONTRIBUTING.md](CONTRIBUTING.md): pull request and quality gate expectations.
