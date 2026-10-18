# Architecture and source locations

## Directory layout

| Path                         | Content                                                                                       |
|------------------------------|-----------------------------------------------------------------------------------------------|
| `Source/aweXpect.Xml`        | The library and the only shipped package                                                      |
| `Tests/aweXpect.Xml.Tests`   | Unit tests (TUnit, `net8.0;net10.0;net48`)                                                    |
| `Tests/aweXpect.Xml.Api.Tests` | Public API snapshots (TUnit, `net8.0`)                                                      |
| `Tests/Aot/aweXpect.Xml.Aot` | Native AOT smoke tests as a console application (`net10.0`)                                   |
| `Benchmarks/aweXpect.Xml.Benchmarks` | BenchmarkDotNet benchmarks                                                            |
| `Pipeline`                   | Fallout build targets, started by `build.ps1`, `build.sh` and `build.cmd`                     |
| `Docs/pages`                 | User documentation, published to the documentation site                                       |
| `Docs/agents`                | Guides for authors and agents, not published                                                  |
| `.github`                    | CI workflows, Dependabot and Mergify configuration                                            |

## Library

Public expectations are extension methods in the namespace `aweXpect`, so that they are available with the core `Expect.That`. Each file holds one expectation of a partial class, named `<Class>.<Expectation>.cs`:

| File                                   | Subject     | Expectations                                                     |
|----------------------------------------|-------------|------------------------------------------------------------------|
| `ThatXmlString.*.cs`                   | `string`    | `IsValidXml` (with `.Which(...)` on the root element), `IsValidXmlMatching`, `IsValidXmlMatchingExactly` |
| `XmlExtensions.cs`                     | `string`    | `AsXml()` on `IsEqualTo(...)`, implemented as a string match type |
| `ThatXElement.*.cs`                    | `XElement`  | `Matches`, `MatchesExactly`, `HasAttribute`, `DoesNotHaveAttribute`, `HasElement`, `DoesNotHaveElement` |
| `ThatXDocument.*.cs`                   | `XDocument` | `Matches`, `MatchesExactly` on the root element                  |
| `Results/XmlWhichResult.cs`            | `string`    | The result of `IsValidXml` that offers `Which`                   |

The comparison lives in `Xml/` (namespace `aweXpect.Xml`):

- `XmlComparer` compares two elements by content: names by namespace and local name, attributes in any order, comments and processing instructions ignored, text and CDATA as one text. Differences are reported by path, e.g. `/order/item[2]/@price`. Child elements are paired by local name and position, and their order is checked separately unless it is ignored.
- `MatchesXmlConstraint<TNode>` backs `Matches` and `MatchesExactly`; `IsValidXmlMatching*` are shortcuts for `IsValidXml().Which(root => root.Matches...(...))`. The expected XML is parsed once when the expectation is specified, so an invalid expected value throws an `ArgumentException` for every subject.
- `XmlMatchType` is the `IStringMatchType` behind `AsXml()`.
- `HasNodeConstraint<TNode>` backs `HasAttribute` and `HasElement` and stores the found node, so that nested expectations only run when it exists.
- `XmlOptions` is immutable; each method in `XmlOptionsExtensions` (`IgnoringElementOrder`, `IgnoringMember`, `Ignoring`) returns a copy. A new option must be copied in the internal copy constructor and listed in the "XML options" failure context.
- `XmlMemberToIgnore` and `MembersToIgnoreTracker` decide which paths are ignored and name the members to ignore that matched nothing, with the path that was probably meant.

Helpers in `Helpers/` (namespace `aweXpect.Helpers`):

- `SecureXml` is the only place that parses XML: DTDs are prohibited, no resolver is set and insignificant whitespace is dropped. It also tells a DTD error apart from other parse errors for the failure message.
- `ThrowHelper.ThrowIfNull` validates arguments with `Tracing.WriteException` from aweXpect.Core.
- `Polyfills/` holds attributes that `netstandard2.0` lacks, compiled only below `net8.0`.

## Tests

- Test files mirror the expectation: `ThatXElement.HasAttribute.Tests.cs` contains `ThatXElement.HasAttribute.Tests` as nested classes in the namespace `aweXpect.Xml.Tests`. Options have their own files, e.g. `XmlOptions.IgnoringMember.Tests.cs`.
- Test methods are named `When<Condition>_Should<Outcome>` and use the arrange, `async Task Act()`, assert pattern with `await That(Act).DoesNotThrow()` or `Throws<FailException>().WithMessage(...)`; aweXpect throws its own `FailException` under TUnit.
- `Usings.cs` provides global usings for aweXpect, including `using static aweXpect.Expect`; TUnit adds its own.
- The AOT smoke checks in `Tests/Aot/aweXpect.Xml.Aot/Checks.cs` state for each expectation the passing result or the expected parts of the failure message, and run identically under the JIT and as a Native AOT binary.

## Build settings

- `Directory.Build.props` (root): strong-name signing for all projects.
- `Source/Directory.Build.props`: package metadata, target frameworks, warnings as errors, documentation file, symbol packages, trimming and AOT flags (`net8.0` and up; the older `net8.0` analyzers are disabled) and `Nullable`/`IsExternalInit` polyfill packages. On `netstandard2.0` nullable reference types are annotations only.
- `Tests/Directory.Build.props`: test target frameworks, TUnit executables of the Microsoft.Testing.Platform with the TRX and code coverage extensions, and the full `aweXpect` package; tests are not signed.
- `Tests/Aot/Directory.Build.props`: `PublishAot`, invariant globalization and trimmer and AOT warnings as errors.
- `.editorconfig`: tabs, CRLF line endings, file-scoped namespaces, explicit types instead of `var`, braces always.
