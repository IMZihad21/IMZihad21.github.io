# Stop repeating dotnet ef flags: repository-level defaults with dotnet-ef.json

- Canonical URL: https://imzihad21.github.io/articles/a/stop-repeating-dotnet-ef-flags-repository-level-defaults-with-dotnet-efjson-epk/
- Source URL: https://dev.to/imzihad21/stop-repeating-dotnet-ef-flags-repository-level-defaults-with-dotnet-efjson-epk
- Web View: https://imzihad21.github.io/articles/a/stop-repeating-dotnet-ef-flags-repository-level-defaults-with-dotnet-efjson-epk/
- Published: 2026-09-30T04:00:48.000Z
- Modified: 2026-09-30T04:00:48.000Z
- Reading time: 6 minutes
- Tags: dotnet, efcore, csharp, database

A multi-project solution forces every `dotnet ef` invocation to carry the same `--project`, `--startup-project`, `--framework`, and `--context` arguments. Those arguments end up in READMEs, shell aliases, CI scripts, and IDE run configurations, and each copy drifts on its own. When one copy omits `--project`, the command targets whatever project is in the current directory, and migration files are written into the wrong project.

Starting with EF Core 11, `dotnet ef` loads default option values from `.config/dotnet-ef.json`. The tool walks up from the current working directory, uses the nearest matching file, and applies its values only to options missing from the command line. One committed file defines the repository's defaults, and an explicit command-line option always wins.

### The problem and production context

The `dotnet ef` commands operate on a target project and a startup project. The target project is where commands add or remove files. The startup project is the one the tools build and run, because they execute application code at design time to read the connection string and model configuration. In a layered solution these are usually different projects, so both must be specified on every call.

- **Failure scenario**: A developer runs `dotnet ef migrations add AddOrders` from the API project directory without `--project`. Per the tool documentation, the default target is the project in the current directory, so the migration files are added to the API project instead of the infrastructure project.
- **Why default approaches fall short**: The defaults come from the current working directory. Every option that differs from the default must be restated by every caller, and nothing verifies that the callers agree.
- **Production impact**: CI pipelines, deployment scripts, and developer machines invoke the tool with different argument sets, so the same command can hit different targets depending on the caller. The discrepancy shows up when a migration lands in the wrong place or a multi-context project fails because `--context` was omitted.

### Mental model and core concepts

The feature adds a file lookup that supplies defaults for options the caller left out. Defaults move from each caller to a file committed with the repository.

#### 1. Discovery

When `dotnet ef` runs, it walks up the directory tree from the current working directory and uses the first `.config/dotnet-ef.json` it finds. A file at the repository root therefore applies from any subdirectory, and a file in a nested directory shadows one further up, because only the nearest file is used. The documentation describes no merging across levels.

#### 2. Precedence

For each supported option, the resolved value is the command-line value when one is supplied, otherwise the configuration value, otherwise the tool's built-in default. Explicit command-line options always take precedence over the file.

The `--context` option needs care because the tool must distinguish "the user passed a context" from "the file supplied one". During review of the implementation, detection of an explicit context was extended to the `--context Foo`, `--context=Foo`, `--context:Foo`, and `-c:Foo` forms. A related review fix ensured that a context injected from the file does not change the skip-optimization behavior of `dbcontext optimize`.

#### 3. Path resolution

The `project` and `startupProject` properties accept relative or absolute paths. Relative paths are resolved against the parent of the `.config` directory that contains the file, not against the working directory. With the file at `<repo>/.config/dotnet-ef.json`, `"project": "src/App.Infrastructure"` resolves to `<repo>/src/App.Infrastructure` whether the command runs from the repository root or from `src/App.Api`.

#### 4. Validation

The file is a JSON object whose properties are all optional. The supported set is `project`, `startupProject`, `framework`, `configuration`, `context`, `runtime`, `verbose`, `noColor`, and `prefixOutput`. String properties take strings and the last three take booleans. The implementation reports errors for malformed or unreadable files and for unsupported properties. During review, a specific message for some options was dropped in favor of one generic message. The exact message wording may change between builds.

### Production implementation

Commit one configuration file at the repository root and drop the repeated flags from every script.

```text
<repository root>/
├── .config/
│   └── dotnet-ef.json
└── src/
    ├── App.Api/
    └── App.Infrastructure/
```

```json
{
  "project": "src/App.Infrastructure",
  "startupProject": "src/App.Api",
  "framework": "net11.0",
  "configuration": "Debug",
  "context": "AppDbContext"
}
```

Before, every caller carried the full argument set:

```bash
dotnet ef migrations add AddOrders \
  --project src/App.Infrastructure \
  --startup-project src/App.Api \
  --framework net11.0 \
  --configuration Debug \
  --context AppDbContext
```

After, the same command runs identically from the repository root or any subdirectory:

```bash
dotnet ef migrations add AddOrders
```

An explicit option overrides a single default without editing the file, for example when a CI job builds `Release`:

```bash
dotnet ef migrations has-pending-model-changes --configuration Release
```

Output behavior can be set the same way with `runtime`, `verbose`, `noColor`, and `prefixOutput`. Set those only where every caller wants them, because a shared file affects every developer and pipeline.

### Architectural trade-offs and edge cases

- **Latency versus consistency**: The lookup adds a directory walk and a file read per invocation, and the documentation gives no measurement of its cost. In return, defaults are defined once and versioned with the code, so the target project no longer depends on the caller's working directory. The cost is implicit behavior: a command line that looks incomplete is now valid, so readers must know the file exists.
- **Failure recovery**: The file is read at startup and holds no runtime state, so a corrupted or stale file is fixed by editing it and rerunning. A malformed or unreadable file is reported as an error, and the implementation review shows unsupported properties are rejected instead of ignored. Confirm the exit code behavior in your own pipeline before using it as a gate.
- **Scale limitations**: Only the nearest file is used, and there is a single `context` default. A repository with several `DbContext` types or several independent solutions cannot express per-context defaults in one file. Put separate files under each subtree and pass `--context` explicitly where the default does not apply. The file covers only the nine listed properties. Connection strings, output directories, and arguments forwarded to the application after `--` stay on the command line.

### Common anti-patterns and gotchas

- **Committing absolute paths**: Absolute paths are accepted, so people paste whatever their machine resolved. The file then fails on every other machine and in CI, where the checkout location differs. Use paths relative to the directory that contains `.config`.
- **Setting a repository-wide `context` in a multi-context repository**: Teams set it to stop typing `--context`. Commands meant for another context then target the wrong one unless the flag is passed. For `dbcontext scaffold`, the documented `--context` names the context class to generate, not an existing one, so check how the configured default interacts with scaffold before setting it, or leave `context` unset in repositories that scaffold.
- **Enabling boolean output flags in the shared file**: Someone sets `verbose: true` or `noColor: true` to match their terminal, and the setting applies to every caller. The documentation lists no negating command-line flag for these options, so an individual cannot turn it off per invocation. Keep personal output preferences out of the committed file unless the whole team wants them.
- **Leaving stale flags in wrapper scripts**: Wrapper scripts keep their old flags after the file is added. Explicit options win, so the scripts keep their old values, and the file is bypassed whenever the two diverge. Remove the redundant flags so the file is the single source.

### Implementation checklist

1. Confirm the installed tools support the feature: run `dotnet ef --version` and update with `dotnet tool update --global dotnet-ef` if the version predates EF Core 11.
2. Create `.config/dotnet-ef.json` at the repository root with only the properties every caller shares, using paths relative to the repository root, and commit it.
3. Run `dotnet ef dbcontext info` from the repository root and again from `src/App.Api`, and verify both invocations report the same target.
4. Run one command with an explicit override, such as `--configuration Release`, and verify the command-line value is used instead of the file's value.
5. Introduce a deliberately malformed file in a scratch branch and verify that the CLI reports an error and that your pipeline step fails.
6. Remove the redundant `--project`, `--startup-project`, `--framework`, and `--context` flags from scripts, READMEs, and CI definitions.
7. Run `dotnet ef migrations has-pending-model-changes` in CI from a subdirectory to verify the file is discovered outside the repository root.

### References

- Microsoft Learn: [EF Core tools reference (.NET CLI), Configuration file](https://learn.microsoft.com/en-us/ef/core/cli/dotnet#configuration-file)
- Documentation source: [dotnet/EntityFramework.Docs, `entity-framework/core/cli/dotnet.md`](https://github.com/dotnet/EntityFramework.Docs/blob/main/entity-framework/core/cli/dotnet.md)
- Implementation: [dotnet/efcore pull request 37966, Add dotnet-ef JSON config defaults and validation features](https://github.com/dotnet/efcore/pull/37966)
- Originating issue: [dotnet/efcore issue 35231](https://github.com/dotnet/efcore/issues/35231), referenced by the pull request