# AutoIntegration.jl

[![CI](https://github.com/projectured/AutoIntegration.jl/actions/workflows/CI.yml/badge.svg)](https://github.com/projectured/AutoIntegration.jl/actions/workflows/CI.yml)

AutoIntegration loads an installed package when the packages that it names as
its triggers are loaded, as the settings of the environment of the user choose.

A package that joins two other packages, such as a plot recipe for a data type
or a backend for a framework, is an *integration*. AutoIntegration lets a user
install integrations as ordinary packages, by name, and then choose for each one
whether it loads by itself or only when a `using` line names it.

## Why

Julia has two ways to ship an integration, and each one fails a user:

- **A package extension** lives in its parent package, and Pkg never installs its
  triggers. An extension of a framework can load only what the framework itself
  installs, so the framework must install every integration and every package
  that they join. An extension inside the integration makes the integration
  install nothing that it joins, so `using TheIntegration` alone fails.
- **An ordinary package** installs what it needs, but it loads only when a
  `using` line names it. A user must name each integration in each session.

Pkg has no optional dependencies, so a package that must work after `add` and
`using` alone holds its dependencies in `[deps]`. AutoIntegration keeps that,
and adds the automatic load on top, as a choice of the user:

- An install is always explicit. A package installs only what it needs.
- To install and to load are two steps. An install never changes what loads.
- Users want different things. One user wants the integrations to load by
  themselves. Another user names each package, because an automatic load costs
  time and memory and can surprise. With many integrations, one switch for all
  is too coarse. So the setting is per package, in the environment of the user,
  and the package gives the default.

## Declare the triggers of a package

A package takes part with a table in its `Project.toml`. Each trigger is a
package name with its uuid, as in `[weakdeps]`. A trigger need not be a
dependency of the package.

```toml
[auto-integration]
default = "auto"

[auto-integration.triggers]
Projectured = "92922de3-b970-4d9a-8b2a-9d6f361397b5"
SimpleDirectMediaLayer = "98e33af6-2ee5-5afd-9e75-cbc738b767c4"
```

`default` is `"auto"` or `"manual"`. A table with no `default` gives `"manual"`.
A package with no table never loads by AutoIntegration. Pkg keeps the table
through `add`, `develop` and `resolve`, and a registry accepts it.

A framework that wants this loads AutoIntegration in its own entry module, for
example with `import AutoIntegration`. Nothing else is needed: each integration
declares its own triggers, and AutoIntegration names no package.

## Set the state of a package

The file `LocalPreferences.toml` beside the `Project.toml` of an environment holds
the setting:

```toml
[AutoIntegration]
ProjecturedSDL = "auto"
ProjecturedDataFrames = "manual"
```

`"auto"` loads the package when its triggers are loaded. `"manual"` loads it only
when a `using` line names it. A package with no entry has the state of its
default. The first environment of the load path that has an entry decides, so
the active project comes first.

`set_auto_integration!` writes the file of the active project. A `using` line in
`Main` reaches only the packages that you added, so add AutoIntegration by name
first:

```
pkg> add AutoIntegration

julia> using AutoIntegration
julia> set_auto_integration!("ProjecturedDataFrames", :manual)   # :auto, :manual, or nothing
```

`nothing` removes the entry, so the default of the package applies. A setting
applies from the next load of a package.

AutoIntegration reads the file itself, with the TOML standard library. Julia
gives the preferences of a package only to an environment that names the package
in `[deps]` or `[extras]`. The environment of a user names the framework, not
AutoIntegration, so Preferences.jl would not see the table.

## How it works

**The candidates.** A candidate is a direct dependency of an environment of the
load path that declares the table. That is the set that a `using` line in `Main`
can reach, so AutoIntegration loads nothing that the user did not install by
name. A folder of packages, such as `@stdlib`, gives no candidate.
AutoIntegration reads the `Project.toml` of each direct dependency, and keeps
the candidates with the times of change of the project files and of the
manifests beside them. So an `add`, an `update` or an `activate` in the session
gives new candidates at the next load.

**The hook.** `__init__` adds one callback to `Base.package_callbacks`. In a
process that writes a cache file, it adds nothing, so no cache file depends on
what one user installed. Julia calls the callback after each load of a package.

**The wait.** The callback does nothing while a load is in progress. Julia calls
the callback of a dependency while the package that imports it still loads. A
candidate loaded then could need the package in progress and load it a second
time, which Julia stops with `ConcurrencyViolationError`. Julia calls the
callback of the outer package after its load ends, and that callback does the
work.

**The load.** The callback loads each candidate that is not loaded, whose
triggers are all loaded and whose state is `"auto"`, with
`Base.require(::Base.PkgId)`. It repeats until a pass loads nothing, because a
loaded package can be the trigger of another. A flag stops a second call while
one call runs, because a load inside the callback calls the callbacks again.

**The result.** A loaded candidate binds no name in `Main`: a user who writes a
name of it adds a `using` line. If a candidate fails to load, a warning names
it, the `using` line of the user goes on, and the session does not try the
candidate again.

## A user of it

[ProjecturEd](https://github.com/projectured/projectured-julia) uses
AutoIntegration. Its umbrella package `Projectured` loads it. Each domain and
backend declares the trigger `Projectured`, and each integration declares
`Projectured` and the package that it joins, for example `ProjecturedSDL` with
`SimpleDirectMediaLayer`.

## Limits

- AutoIntegration does not unload a package that is set to `"manual"` after it
  loaded.
- The callback reads `Base.package_locks` under `Base.require_lock` to know
  whether a load is in progress. That is a name of Julia that is not public.

## Licence

MIT, in `LICENSE`.
