---
name: Generate a Refit client from an OpenAPI specification
description: >-
  Turn an OpenAPI 2.0/3.x specification into compiling C# Refit interfaces and
  contracts using the Refitter CLI, an MSBuild task, or the Roslyn source
  generator — and pick the right one for the project.
api: cli/refitter-cli.yml
tool: refitter
operations: []
commands:
  - refitter [URL or input file] [OPTIONS]
generated: '2026-08-06'
method: generated
source: https://refitter.github.io/articles/cli-tool.html
---

# Generate a Refit client with Refitter

Refitter reads an OpenAPI 2.0 (Swagger) or 3.x document and emits C# `Refit`
interfaces plus their contract types. Three distribution forms consume the same
`.refitter` settings file, so choosing one is a build-integration decision, not a
feature decision.

## 1. Pick the distribution form

| Situation | Use | Package |
|---|---|---|
| One-off or scripted generation, output checked into the repo | CLI tool | `Refitter` (`dotnet tool install --global Refitter`) |
| Generation on every build, output on disk, full MSBuild control | MSBuild task | `Refitter.MSBuild` |
| Generation at compile time, no files on disk | Source generator | `Refitter.SourceGenerator` |
| CI without a .NET SDK on the runner | Container | `docker pull christianhelle/refitter` |

Since v2.0.0 the source generator emits code **in-memory** via Roslyn
`AddSource()` — it no longer writes `.g.cs` files, and `Refit` no longer flows
transitively from the source generator package. If a project version-controls
generated files or references them from build scripts, use the CLI or the
MSBuild task instead.

## 2. Generate

Minimum viable invocation:

```shell
refitter ./openapi.json --namespace "Acme.Api.Generated" --output ./GeneratedCode.cs
```

Straight from a live spec URL:

```shell
refitter https://petstore3.swagger.io/api/v3/openapi.yaml
```

## 3. Prefer a `.refitter` settings file over flags

Once more than two or three options are in play, move them into a `.refitter`
file and check it in — the CLI, the MSBuild task and the source generator all
read it, so the three forms stay in sync.

```shell
refitter ./openapi.json --settings-file ./openapi.refitter --output ./GeneratedCode.cs
```

`--settings-file` **ignores every other flag except `--output`**. Do not mix the
two and expect the flags to win.

The settings file schema is published at
`json-schema/refitter-refitter-file-schema.json` (upstream
`https://github.com/christianhelle/refitter/blob/main/docs/json-schema.json`);
point an editor at it for completion and validation.

## 4. Shape the output deliberately

- `--multiple-interfaces ByTag` (or `ByEndpoint`) splits one giant interface into
  per-tag or per-endpoint interfaces. On a large spec this is almost always what
  you want.
- `--multiple-files` splits the output into `RefitInterfaces.cs`,
  `DependencyInjection.cs` and `Contracts.cs`.
- `--tag Pet --tag Store` and `--match-path '^/pet/.*'` narrow generation to the
  slice of the API you actually call. Repeat either flag; tags are OR'ed.
- `--trim-unused-schema` drops component schemas nothing references. Pair it with
  `--keep-schema '^Model$'` when a type is only reached reflectively.
- `--use-api-response` returns `Task<IApiResponse<T>>` instead of `Task<T>` — take
  this when you need status codes and headers, not just the deserialized body.
- `--cancellation-tokens` adds a trailing `CancellationToken` to every method.
- `--immutable-records` emits contracts as records rather than classes.

## 5. Treat the input specification as untrusted

This is the operating rule that matters most, and it comes from the project's own
published advisories (see `security/refitter-vulnerability-disclosure.yml`):
four advisories in June 2026 — one high, three critical — were all
attacker-controlled-OpenAPI attacks against the generator, where crafted paths,
content-type keys or header parameter names escaped into Refit attributes and
executed at assembly load.

- **Pin to 2.1.2 or later.** The RCE class is fixed there; the `$ref` SSRF/LFI
  class is fixed in 2.1.1.
- **Leave `--allow-remote-refs` off** unless you control every host the document
  can reach. Remote `$ref` resolution is disabled by default precisely because it
  was a generation-time SSRF vector.
- Generating from a spec you did not author is the threat model. Review the
  generated file before it compiles into a shipping assembly.

## 6. Verify

The generated file is ordinary C# — the only real check is that it compiles and
that the interface shape matches the API you meant to call:

```shell
dotnet build
```

Refitter's own smoke tests do exactly this: generate against dozens of specs
(v2.0, v3.0, v3.1, v3.4) and compile every result against a console app.

## 7. Telemetry

Refitter collects anonymous usage telemetry and error reports. Disable it with
`--no-logging` (the flag also suppresses error logging). The
`--telemetry-source`, `--telemetry-file-count` and `--telemetry-runtime` flags
are internal to the MSBuild integration — the docs note they are self-reported
and advisory, so do not treat them as proof of provenance.

## Related

- CLI surface: `cli/refitter-cli.yml`
- Release history and breaking changes: `changelog/refitter-changelog.yml`
- Version/deprecation posture: `lifecycle/refitter-lifecycle.yml`
- Standards consumed: `conformance/refitter-conformance.yml`
