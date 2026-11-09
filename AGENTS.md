# bus-dotnet — working in this repo

Transport-agnostic messaging contracts for .NET — messages, read-only message contexts and send/publish buses — shipped on NuGet as `JorgeCostaMacia.Bus`, the root every `JorgeCostaMacia.Bus.*` transport implements. The transports live in their own repositories, [bus-kafka-dotnet](https://github.com/JorgeCostaMacia/bus-kafka-dotnet) and [bus-rabbitmq-dotnet](https://github.com/JorgeCostaMacia/bus-rabbitmq-dotnet), so a patch to one transport republishes neither the other nor these contracts.

## Layout

- `src/JorgeCostaMacia.Bus/` — the package: everything under `Domain/` (`Buses/`, `Contexts/`, `Messages/`). `test/JorgeCostaMacia.Bus.Tests/` — its tests, mirroring those folders. `assets/` — icons + social preview.
- **3-tier `Directory.Build.props`**: **root** (repo identity — Authors / Company / Copyright / Repository — + the single `VersionPrefix`; TFM `net10.0`; ImplicitUsings, Nullable, AnalysisLevel, EnforceCodeStyleInBuild, TreatWarningsAsErrors, the NuGet audit; `IsPackable` false by default) → **`src/`** (`IsPackable` true, package-output: icon / readme / license, SourceLink, symbols, `GenerateDocumentationFile`, pack of LICENSE/COPYRIGHT/icon/README) → **`test/`** (test settings). The `src` csproj declares **only** `Description` / `PackageTags`; everything else is inherited from the props (don't restate it).

## Architecture — concrete-first transports over agnostic contracts (locked)

```
JorgeCostaMacia.Bus            this repo: transport- AND pattern-agnostic vocabulary — IMessage / ITracedMessage /
│                              IFilteredMessage, ITransport, IContext + envelope facets, ISenderBus<T> /
│                              IPublisherBus<T>, IHandler (NO requester / request-response, NO ICommand/IEvent)
├─ Bus.Kafka                  bus-kafka-dotnet: own implementation on Confluent.Kafka
└─ Bus.RabbitMQ               bus-rabbitmq-dotnet: own implementation on the official RabbitMQ.Client (NOT MassTransit)
```

- **Concrete-first**: the command/event distinction is defined BY EACH TRANSPORT, not here — a transport ships its own `Command` / `Event` records, handler and subscriber bases and `IBus : ISenderBus<Command>, IPublisherBus<Event>`. Code shared across transports types against these contracts (`ITracedMessage`, the facets, `IHandler`), which every transport implements. Don't add `ICommand` / `IEvent` here.
- **No requester** — request/response is not part of the bus. **No query bus.**
- **Ordering is a non-concern by design**: transports partition freely and each consumer decides how it resolves conflicts — typically by a version its data carries, applied only when newer. **`ITracedMessage.AggregateOccurredAt`** is when the event occurred and can serve as that order; a consumer whose data carries a version of its own compares that instead (see the `messaging` skill). `AggregateId` is domain/tracing metadata, **not** a partition key.

## Targets & stack

- TFM: **`net10.0`**, single-target, as every transport is (Quartz 4, which their retry packages use, ships only `lib/net10.0`). **Do not reintroduce a per-TFM conditional.**
- No dependency at all: the contracts are plain interfaces and records, so a transport never inherits a third-party version from here.
- Tests: **xUnit v4 (the `xunit.v3` package, 4.x) on Microsoft.Testing.Platform v2 (MTP)** — test projects are `OutputType=Exe`, `dotnet test` runs MTP because the root **`global.json`** opts in, and `xunit.runner.visualstudio` is referenced for IDE discovery, as `testing-dotnet` sets up. Running them needs the **.NET 10 SDK**.
- Source is **UTF-8 without BOM** (`.editorconfig` `charset = utf-8`). camelCase locals, PascalCase types, I-prefixed interfaces. Copyright year stays **2023** (deliberate — don't bump).
- **Explicit types everywhere — spell the type out.** Never `var`, never target-typed `new()`, never collection expressions `[]`: write `new Foo(...)`, `new byte[] { ... }`, `new List<T> { ... }`, `Array.Empty<T>()`. The `.editorconfig` sets all three to explicit, but only `var` is analyzer-enforced — the analyzer never reports the implicit forms. Develop (`dotnet-build-nuget` from github-workflows) guards target-typed `new(` and implicit `new[]` with a **grep step** that fails the build, and collection expressions `[…]` with a second one (`code-style-dotnet`'s pattern: attributes, indexers, array types and list patterns never match it). Do not introduce any of them.

## Who depends on this

The transports, through the **published** package (`PackageReference`, never `ProjectReference` across repos). A change here therefore releases in two steps: this repo first; then, once nuget.org has indexed it, each transport that needs it bumps its `JorgeCostaMacia.Bus` reference and releases. A transport that does not need the change stays on the version it has. A breaking change here is a major here and in every transport that takes it.

## Dependencies — Central Package Management

Third-party package versions are centralized in **`Directory.Packages.props`** (repo root, `ManagePackageVersionsCentrally=true`): add or bump them **there** as `<PackageVersion>`, and reference packages in csproj **without** a `Version`.

**The strict gates are on**: `TreatWarningsAsErrors` and the NuGet vulnerability audit on every restore (direct and transitive, every severity), so any warning — an advisory included — fails the build. The fix is a version bump or a transitive override in CPM, never `NoWarn`. The root `nuget.config` clears every inherited source and keeps nuget.org alone: this package consumes nothing else, and an inherited second source would raise `NU1507`. It is **deny-read** for agents — hand changes to it to the user.

## CI / publishing

The three workflows call the reusable ones of [github-workflows](https://github.com/JorgeCostaMacia/github-workflows) (`@v1`): `develop.yml` runs `dotnet-build-nuget` on develop and PRs (the explicit-types and collection-expression greps, build, the `dotnet format` check, test, pack — no publish); `main.yml` runs the same build on a push to `main` and then pushes the `nuget` artifact to nuget.org with Trusted Publishing (`NuGet/login`, OIDC) — that job stays here because the nuget.org policy names this repo's `main.yml`; `release.yml` creates the GitHub Release of each `v*` tag. **The (central) `VersionPrefix` is the publish gate**: `--skip-duplicate` leaves an existing version alone, so only a release that moved it publishes.

## Branching & releases — GitFlow

Use the **`gitflow` skill** for any branch/release work — never improvise.

- Feature/bugfix → `feature/`|`bugfix/<name>-<ts>` from develop → finish `--no-ff` into develop.
- Release → `release/<version>` from develop → bump the **single** `VersionPrefix` in the **root** `Directory.Build.props` (the one holding the solution identity) → Release Finish (merge develop+main, annotated tag `v<version>`, atomic push).
- Use git's **default merge message** (`--no-ff --no-edit`, never `-m`).
- Branch prefixes only: `feature` / `bugfix` / `release` / `hotfix`.

## Git etiquette

- Commit under **your own identity** — don't hardcode anyone's name/email.
- Keep history clean — **no** `Co-Authored-By` / AI-assistant trailers in commits or messages.
- Merges use git's **default** message (see *Branching & releases* above).

## Relevant skills

Skills that apply to this repo — let them trigger, or invoke explicitly. The first group is from `jorgecostamacia-agent-skills` (each `-dotnet` skill brings the generic one it realizes); the rest from `dotnet-agent-skills` (the `dotnet/skills` marketplace).

A skill listed here but missing on this machine is one command away: `claude plugin marketplace add JorgeCostaMacia/skills` (or `dotnet/skills`) once, then `claude plugin install <skill>@<marketplace>`.

- **`bus-dotnet`** (with **`messaging`**) — how services use the bus these contracts define: the message records and their two constructors, thin handlers and subscribers, `Send`/`Publish` with the incoming transport, the retry ladder. `messaging` is the why — duplicates and disorder, the version travelling with the data. A change here is a change in what those skills describe: keep them in step.
- **`code-style-dotnet`** — how the C# is written: explicit types, the 120-column width with declarations and constructions vertical, comments wrapped too, the gates.
- **`build-dotnet`** — the three-tier `Directory.Build.props`, the single `VersionPrefix`, CPM, the three-day package age, the NuGet audit as a build gate, the library's packing metadata, and the `nuget.config` base.
- **`testing-dotnet`** (with **`testing`**) — xUnit v3 on Microsoft.Testing.Platform, the test project layout.
- **`clean-architecture`** / **`solid`** — the bus as the messaging seam (consumers are driving adapters), and design review of the public surface.
- **`gitflow`** — all branch/release work.
- **`changelog`** — the `CHANGELOG.md` entry a release adds when it asks something of the packages' consumers (a major, a new requirement, a changed default), written with the version bump.
- **`dotnet`** — C# language server + general .NET development.
- **`dotnet-msbuild`** — `Directory.Build.props`, project-file quality, CPM.
- **`dotnet-nuget`** — dependency management.
- **`dotnet-test`** / **`dotnet-test-migration`** — tests; the xUnit v4 / MTP v2 setup.
- **`dotnet-upgrade`** — target-framework migrations.

Not relevant to this repo (skip): `bus-kafka-dotnet` (the Kafka decisions are bus-kafka-dotnet's), `ddd` / `ddd-dotnet`, `validation-dotnet`, `clean-architecture-dotnet`, `efcore-dotnet` / `efcore-postgres-dotnet`, `logging-dotnet`, `dotnet-aspnetcore` (that's http-dotnet), `dotnet-data`, `dotnet-ai`, `dotnet-maui`, `dotnet-blazor`, `dotnet-template-engine`, `dotnet11`, `dotnet-diag`, `dotnet-advanced`.

## Build & test

```
dotnet format bus-dotnet.slnx                  # apply .editorconfig (using order, whitespace) — run before committing
dotnet build  bus-dotnet.slnx -c Release
dotnet test   bus-dotnet.slnx -c Release       # MTP v2 via global.json (needs the .NET 10 SDK); --logger is VSTest-only (MTP0001)
dotnet pack   bus-dotnet.slnx -c Release        # packs the package; the tests are IsPackable=false
```

The tests need nothing external.

Run **`dotnet format` before committing** — it applies the `.editorconfig` (using ordering, whitespace, line endings), the CLI equivalent of Visual Studio's *Code Cleanup*, and it is the normalizer for a `.cs` a shell tool wrote with LF. Develop checks it after the build with `dotnet format --verify-no-changes --no-restore`: the solution is clean, so the check only ever fails on what the push itself brought. `.gitattributes` checks `*.cs` out as CRLF (`eol=crlf`), so the Linux runner sees what the `.editorconfig` requires.
