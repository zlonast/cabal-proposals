# Formalize a stable public API tier for cabal-install

I would like to present my research on this topic to the Cabal developers and the community

## Summary
Split the ~140 exported modules of `cabal-install` into a small, stability-guaranteed public API and an internal implementation:

* `cabal-install` exposes **5 new modules** — `Distribution.Client.API.{Config,PackageIndex,Project,Build,Download}` — thin entry points designed around the actual needs of the reverse dependencies;
* everything else (147 modules) moves to a new internal package `cabal-install-internal` (following the `cabal-install-solver` precedent), consumed privately by `exe:cabal` and the test-suites.

The design was validated by migrating all four live reverse dependencies onto the new API: all of them build against it, and the migration *reduced* their code by ~210 lines in total.

## Motivation
The cabal-install API is massive and barely used: out of ~140 modules, six reverse dependencies import only 32; meanwhile, PRs like #12289 cannot be backported to releases without breaking this usage. Half of this usage falls within the orchestration layer.

## Current situation

The current situation is such that we don't know when we can release a backport; we would like to be able to release minor versions more frequently that include fixes for cabal-install.

### Hackage package status

| Package | Latest version | Latest release | Age | `cabal-install` bounds | Hackage build |
|---|---|---|---|---|---|
| hackage-revdeps | 0.4.1 | 2026-08-13 | ~1 mo | `>=3.8 && <3.19` | OK |
| cabal-add | 0.2.1 | 2026-07-01 | ~2.5 mos | `>=3.12 && <3.19` | OK |
| cabal-matrix | 1.0.3.0 | 2026-06-08 | ~3 mos | `>=3.8 && <3.17` (+solver) | OK |
| cabal-hoogle | 3.16.0.0 | 2026-01-10 | ~8 mos | `>=3.10 && <3.17` | OK |
| hix | 0.9.1 | 2025-04-19 | ~17 mos | `>=3.16.0.0 && <3.17` (+solver) | build fails |
| guardian | 0.5.0.0 | 2024-02-22 | ~31 mos | no bounds | build fails |

### Usage depth summary

| Package | Seed (out of 32) | Usage pattern |
|---|---:|---|
| hix | 19 | maximum: solver facade (`Dependency`, `SolverInstallPlan`, `Types.AllowNewer`), `Types.*`, `IndexUtils`, `Setup`, `CmdSdist`, `CmdUpdate`, `Upload`, `Types.Credentials`/`Repo` |
| cabal-hoogle | 12 | orchestration layer: `ProjectOrchestration`, `ProjectPlanning(.Types)`, `ScriptUtils`, `CmdBuild`, `CmdErrorMessages`, `TargetProblem`, `NixStyleOptions`, `InstallPlan`, `Setup` |
| guardian | 6 | same orchestration layer: `establishProjectBaseContextWithRoot`, `withInstallPlan`, `CmdUpdate.updateAction` |
| cabal-matrix | 5 | lightweight: `Config`, `GlobalFlags`, `IndexUtils`, `Sandbox.loadConfigOrSandboxConfig`, `Types.SourcePackageDb` |
| cabal-add | 4 | `ProjectConfig` (ProjectFileParser), `RebuildMonad`, `HttpUtils.configureTransport`, `DistDirLayout` |
| hackage-revdeps | 2 | trivial: `Config.loadConfig`, `GlobalFlags.globalCacheDir` — just to locate the Hackage cache directory |

### Seed table: closure and consumers

`total` — size of the individual closure (how many modules the seed pulls in on its own, including itself);
`uniq` — modules unreachable from any other seed.

| Module (`Distribution.Client.`) | total | uniq | Who uses |
|---|---:|---:|---|
| `Config` | 52 | 0 | cabal-matrix, hackage-revdeps, hix |
| `GlobalFlags` | 29 | 0 | cabal-matrix, hackage-revdeps, hix |
| `NixStyleOptions` | 49 | 0 | cabal-hoogle, guardian, hix |
| `Types.SourcePackageDb` | 8 | 0 | cabal-hoogle, cabal-matrix, hix |
| `CmdUpdate` | 93 | 1 | guardian, hix |
| `DistDirLayout` | 53 | 0 | cabal-add, cabal-hoogle |
| `IndexUtils` | 69 | 0 | cabal-matrix, hix |
| `InstallPlan` | 26 | 0 | cabal-hoogle, guardian |
| `ProjectConfig` | 68 | 0 | cabal-add, guardian |
| `ProjectOrchestration` | 92 | 0 | cabal-hoogle, guardian |
| `ProjectPlanning` | 86 | 0 | cabal-hoogle, guardian |
| `Setup` | 41 | 0 | cabal-hoogle, hix |
| `CmdBuild` | 95 | 1 | cabal-hoogle |
| `CmdErrorMessages` | 88 | 0 | cabal-hoogle |
| `CmdSdist` | 94 | 1 | hix |
| `Compat.Prelude` | 2 | 0 | hix |
| `Dependency` | 25 | 0 | hix |
| `Dependency.Types` | 3 | 0 | hix |
| `HttpUtils` | 25 | 0 | cabal-add |
| `IndexUtils.Timestamp` | 3 | 0 | hix |
| `ProjectFlags` | 44 | 0 | hix |
| `ProjectPlanning.Types` | 60 | 0 | cabal-hoogle |
| `RebuildMonad` | 16 | 0 | cabal-add |
| `Sandbox` | 78 | 2 | cabal-matrix |
| `ScriptUtils` | 93 | 0 | cabal-hoogle |
| `SolverInstallPlan` | 17 | 0 | hix |
| `TargetProblem` | 87 | 0 | cabal-hoogle |
| `Types` | 16 | 0 | hix |
| `Types.AllowNewer` | 3 | 0 | hix |
| `Types.Credentials` | 1 | 0 | hix |
| `Types.Repo` | 5 | 0 | hix |
| `Upload` | 56 | 2 | hix |

## Separating active and abandoned packages

### Sensitivity of the closure to the set of rdeps

`guardian` (dead, no release for 2.5 years) and `hix` (locked to `3.16.x`, build fails on Hackage)
do not track recent releases — recalculating without them reveals the "cost" of supporting them
in the public API:

| Set | Seeds | Public tier | Internal |
|---|---:|---:|---:|
| all 6 rdeps | 32 | 101 | 46 |
| excluding `guardian` and `hix` | 19 | **97** | **50** |
| + excluding `cabal-hoogle` | 9 | **78** | **69** |

### Stable API core: 19 modules

Changes can only be rolled out in a new major version of `cabal`, so it makes sense to start
defining the stable API based on direct imports from the four active rdeps (excluding `guardian` and `hix`)—
this amounts to exactly **19 modules**, grouped below. The "seed" modules here are the primary
candidates for the public tier; the ~97 closure modules depending on them are secondary and
can gradually be moved to `internal`.

Note that in the final design these 19 modules are not published as-is: they determine the surface of the new wrapper modules and become private dependencies of cabal-install-internal

#### User configuration (base layer)

| Module | Contents | Users |
|---|---|---|
| `Distribution.Client.Config` | `SavedConfig`, `loadConfig` — loading/saving `~/.cabal/config` (remote-repo, jobs, proxy…); entry point for almost any tool | cabal-matrix, hackage-revdeps |
| `Distribution.Client.GlobalFlags` | `GlobalFlags(..)`, `globalCacheDir` — global cache/log paths, CLI-wide flags | cabal-matrix, hackage-revdeps |

#### Package index

| Module | Contents | Users |
|---|---|---|
| `Distribution.Client.IndexUtils` | `getSourcePackages`, `getInstalledPackages` — reading the local Hackage index (`01-index.tar`), timestamps, active repos | cabal-matrix |
| `Distribution.Client.Types.SourcePackageDb` | `SourcePackageDb(..)`, `lookupPackageName` — database of all source packages from the index | cabal-hoogle, cabal-matrix |

#### Project and Layout

| Module | Contents | Users |
|---|---|---|
| `Distribution.Client.ProjectConfig` | `ProjectConfig(..)`, `ProjectRoot(..)`, parsing `cabal.project` (cabal-add also uses `ProjectFileParser`) | cabal-add |
| `Distribution.Client.DistDirLayout` | `DistDirLayout(..)`, `distBuildDirectory` — `dist-newstyle` layout, build artifact paths | cabal-add, cabal-hoogle |

#### Build Orchestration (10 modules — the "blob" core)

| Module | Contents | Users |
|---|---|---|
| `Distribution.Client.NixStyleOptions` | `NixStyleFlags(..)`, `defaultNixStyleFlags` — common set of flags for nix-style commands (config+install+project flags) | cabal-hoogle |
| `Distribution.Client.Setup` | Command flag parsers/types: `InstallFlags`, `configCompilerAux'`, `withRepoContext`, `IsCandidate` | cabal-hoogle |
| `Distribution.Client.ProjectOrchestration` | Command phases: discovery → planning → build; `establishProjectBaseContext(WithRoot)`, `withInstallPlan`, `CurrentCommand` | cabal-hoogle |
| `Distribution.Client.ProjectPlanning` | Constructing the elaborated install plan: `ElaboratedInstallPlan`, `ElaboratedConfiguredPackage` | cabal-hoogle |
| `Distribution.Client.ProjectPlanning.Types` | Plan types: `elabDistDirParams`, unit IDs, component settings | cabal-hoogle |
| `Distribution.Client.InstallPlan` | Installation graph: `GenericPlanPackage`, `depends`, topological traversals | cabal-hoogle |
| `Distribution.Client.ScriptUtils` | Helpers for project target commands: `withContextAndSelectors`, script-mode handling | cabal-hoogle |
| `Distribution.Client.TargetProblem` | `TargetProblem(..)` — target selection error type + rendering | cabal-hoogle |
| `Distribution.Client.CmdBuild` | `cabal build` implementation: `buildAction`, `BuildFlags`, target selectors | cabal-hoogle |
| `Distribution.Client.CmdErrorMessages` | Error rendering: `reportTargetProblems`, `renderCannotPruneDependencies` | cabal-hoogle |

#### Infrastructure

| Module | Contents | Users |
|---|---|---|
| `Distribution.Client.HttpUtils` | `DownloadResult`, `configureTransport` — HTTP transport selection (HTTP/hackage-security), proxies | cabal-add |
| `Distribution.Client.RebuildMonad` | `Rebuild` monad, `runRebuild` — file change tracking, basis for file monitors | cabal-add |

#### Legacy

| Module | Contents | Users |
|---|---|---|
| `Distribution.Client.Sandbox` | `loadConfigOrSandboxConfig` — compatibility with old sandboxes, built on top of `loadConfig` | cabal-matrix |

## Proposed Change

### Target design: a new public API over a private implementation

Breaking changes can only be rolled out in a new major version of Cabal, so the
target architecture is as follows: **several new wrapper modules become public**,
while all 147 existing modules move into a separate internal package,
`cabal-install-internal`, accessed directly by `exe:cabal`
(precedent: the `cabal-install-solver` split in Cabal 3.10).

### New public modules (Draft)

```
Distribution.Client.API.Config
Distribution.Client.API.PackageIndex
Distribution.Client.API.Project
Distribution.Client.API.Build
Distribution.Client.API.Download
```

### Diagram: Who Needs What

```
API.Config  API.PackageIndex  API.Project  API.Build  API.Download
hackage-revdeps      ●
cabal-matrix         ●              ●
cabal-add            ●                             ●
cabal-hoogle         ●              ●              ●            ●
-----------------------------------------------------------------------------------
almost everyone      ●   ← common core; the rest — on an as-needed basis
```

### Mapping: What Each New Module Covers

| New Module | Replaces | Consumer |
|---|---|---|
| `API.Config` | `Config`, `GlobalFlags` (+removes `Sandbox`) | hackage-revdeps; cabal-matrix |
| `API.PackageIndex` | `IndexUtils`, `Types.SourcePackageDb` | cabal-matrix |
| `API.Project` | `ProjectConfig`, `DistDirLayout`, `RebuildMonad`, `HttpUtils` | cabal-add |
| `API.Build` | `ProjectOrchestration`, `ProjectPlanning(.Types)`, `InstallPlan`, `ScriptUtils`, `TargetProblem`, `CmdBuild`, `CmdErrorMessages`, `NixStyleOptions`, `Setup` | cabal-hoogle |
| `API.Download` | `HttpUtils` | reserved (no current consumer: `configureTransport` moved inside `projectLocalCabalFiles`) |

### Design Principles

1. **Existing modules remain unchanged**
2. **Custom types instead of internal ones**

## Migrating reverse dependencies to the new API

I’ve downloaded the packages and created the patches. I’ll upload them once this proposal is approved.

### Replacing internal imports with the API

| Package | Previous (cabal-install modules) | New |
|---|---|---|
| hackage-revdeps | `Config`, `GlobalFlags` | `API.Config` (1 import) |
| cabal-matrix | `Config`, `GlobalFlags`, `IndexUtils`, `Sandbox`, `Types.SourcePackageDb` + `Solver.Types.PackageIndex` | `API.Config` + `API.PackageIndex`; dependency on `cabal-install-solver` removed entirely |
| cabal-add | `DistDirLayout`, `HttpUtils`, `ProjectConfig` (+`ProjectFileParser`), `RebuildMonad` | `API.Project.projectLocalCabalFiles` (1 call instead of 6 imports); custom parsing library left untouched |
| cabal-hoogle | `CmdBuild`, `CmdErrorMessages`, `DistDirLayout`, `InstallPlan`, `NixStyleOptions`, `ProjectOrchestration`, `ProjectPlanning(.Types)`, `ScriptUtils`, `Setup`, `TargetProblem`, `Types.SourcePackageDb` | `API.Build` (`buildTargetsAndDirs` + `runBuild` + `BuildOptions`); reading `LocalBuildInfo` via the public Cabal API |

To achieve this, the `API.*` modules were extended based on actual requirements (`Project.projectLocalCabalFiles`,
`Build.{BuildOptions, runBuild, buildTargetsAndDirs, BuildTarget}`); the resulting size of the new
public layer is 5 modules, totaling **462 lines**.

### Scope of changes by package

| Package | Files | +Lines | −Lines | Note |
|---|---:|---:|---:|---|
| hackage-revdeps | 3 | +12 | −16 | + new `cabal.project` (11 lines) |
| cabal-matrix | 2 | +15 | −20 | |
| cabal-add | 2 | +6 | −61 | `app/Main.hs`: −58 lines — all discovery machinery moved to the API |
| cabal-hoogle | 5 | +59 | −205 | `Common.hs` −146: `buildAction` fork no longer needed |
| **Total** | **12** | **+92** | **−302** | migration **reduced** consumer code by ~210 lines |

### Independent breaking changes in Cabal/cabal-install 3.19 (discovered during migration)

| # | Change | Affected packages | Solution |
|---|---|---|---|
| 1 | `Distribution.Verbosity`: `silent`/`normal` became `VerbosityFlags`; full `Verbosity` is constructed via `mkVerbosity defaultVerbosityHandles <flags>` | hackage-revdeps, cabal-matrix, cabal-hoogle | local `defaultVerbosity`/`silentVerbosity` wrapper |
| 2 | `Distribution.Make` module removed from Cabal (`Make` build-type no longer supported) | cabal-hoogle (`act-as-setup`) | `Make` branch → explicit "no longer supported" error |
| 3 | `resolveTargets'` → `resolveTargetsFromSolver`; `reportTargetProblems` moved to private `reportBuildTargetProblems` | cabal-hoogle (3.16 code) | handled by `API.Build.buildTargetsAndDirs` |
| 4 | Signatures for `readProjectConfig`/`findProjectPackages`/`ProjectFileParser` changed between 3.16 and 3.19 (cabal-add used CPP branches) | cabal-add | CPP removed — superseded by `API.Project.projectLocalCabalFiles` |
| 5 | Sandbox-era API (`loadConfigOrSandboxConfig`) — not part of the public layer | cabal-matrix | `API.Config.loadGlobalConfig` + `API.PackageIndex.withPackageIndex` |

Conclusion: the public `API.*` layer covers all four active consumers; migration simply
involves replacing imports and, on average, **reduces** their code size. All independent breaking changes in 3.19
are concentrated in the 5 points above — candidates for the new major version's ChangeLog.

### Build status

| Package | Build | Smoke test |
|---|---|---|
| hackage-revdeps | ✅ | `--help` works |
| cabal-matrix | ✅ | `--help` works |
| cabal-add | ✅ | `--help` works |
| cabal-hoogle | ✅ | built and linked |

## Alternatives Considered
1. **Do nothing; backport freely.**
2. **Physical `src`/`src-internal` split inside one package.**

## Backwards Compatibility / Migration
1. Explicit separation into `cabal-install` and `cabal-install-internal`.
2. I will send pull requests with the changes to everyone.

## Interested parties
Maintainers of all six reverse dependencies: Bodigrim (`cabal-add`, `hackage-revdeps`), kokobd (`cabal-hoogle`), mniip (`cabal-matrix`), tek (`hix`), HiromiIshii/deepflowinc (`guardian`) — plus the Cabal maintenance team.
`Have you contacted them?` Nope

## Implementation Notes
Yes, I will implement this myself in time for the next release.

## Open Questions
1. `API.Download` currently has no consumer
2. Criteria for adding new modules to the public tier 
3. Is cabal-install-internal published on Hackage? No

## References
1. Can't backport #12289
2. Split like `cabal-install-solver`

