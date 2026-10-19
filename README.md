# Julia Package Development Guide

**Package:** `{{package_name}}` · **Repository:** https://github.com/{{owner}}/{{package_name}}

Requires [Julia](https://julialang.org/install/) 1.12 or later and [Git](https://git-scm.com/).
Run every command from the repository root. Collaboration follows [ColPrac](https://github.com/SciML/ColPrac).

## Setup

Once per machine: install [Revise.jl](https://github.com/timholy/Revise.jl) and [Runic.jl](https://github.com/fredrikekre/Runic.jl) into shared environments, so they never enter `Project.toml`.

```sh
julia --startup-file=no --project=@revise -e 'using Pkg; Pkg.add("Revise")'
julia --startup-file=no --project=@runic  -e 'using Pkg; Pkg.add("Runic")'
```

Once per package: clone and install the dependencies. Contributors clone their [fork](https://github.com/{{owner}}/{{package_name}}/fork) instead and add this repository as `upstream`.

```sh
git clone https://github.com/{{owner}}/{{package_name}}.git
cd {{package_name}}
julia --startup-file=no --project=. -e 'using Pkg; Pkg.instantiate()'
julia --startup-file=no --project=docs -e 'using Pkg; Pkg.instantiate()'
```

## Develop

1. Branch from `main`, named after the issue. Open an [issue](https://github.com/{{owner}}/{{package_name}}/issues) first for significant changes.
2. Start Julia with Revise; edits under `src/` apply to the running session.
3. Edit the code together with its docstrings, the tests under `test/`, and the `## [Unreleased]` section of `CHANGELOG.md`.
   Add dependencies with Pkg; the `[compat]` entry is written automatically, widen it if older versions work.
   Test-only and docs-only dependencies go to `--project=test` and `--project=docs`.
4. Format.
5. Test.
6. Build the docs and open `docs/build/index.html`.
7. Commit and open a pull request once steps 4 to 6 pass.

Start (steps 1 and 2).

```sh
git switch -c issue/123
julia --project=. -i -e 'insert!(LOAD_PATH, 2, "@revise"); using Revise; using {{module_name}}'
```

Add a dependency (step 3), only when needed.

```sh
julia --startup-file=no --project=. -e 'using Pkg; Pkg.add("SomePackage")'
julia --startup-file=no --project=. -e 'using Pkg; Pkg.compat("SomePackage", "1.2")'
```

Check and ship (steps 4 to 7).

```sh
julia --startup-file=no --project=@runic -m Runic --inplace .
julia --startup-file=no --project=. -e 'using Pkg; Pkg.test()'
julia --startup-file=no --project=docs docs/make.jl
git add src test docs CHANGELOG.md
git commit -m "Add fast solver (#123)"
git push -u origin issue/123
```

## Release (maintainers)

Versions follow [SemVer](https://semver.org/) and the [ColPrac guidance on releases](https://github.com/SciML/ColPrac#guidance-on-package-releases).

1. In one pull request `Release vX.Y.Z`, set `version` in `Project.toml`, rename `## [Unreleased]` in `CHANGELOG.md` to `## [X.Y.Z] - YYYY-MM-DD` with a fresh `## [Unreleased]` above it, and set `version` and `date-released` in `CITATION.cff`.
2. Comment on the merge commit:
   ```
   @JuliaRegistrator register
   ```
   [Registrator](https://github.com/JuliaRegistries/Registrator.jl) registers the version in [General](https://github.com/JuliaRegistries/General), and [TagBot](https://github.com/JuliaRegistries/TagBot) tags the release.

First release only: install the [Registrator GitHub App](https://github.com/apps/juliateam-registrator) on the repository.

## About this guide

Maintained at https://github.com/JuliaPackageFactory/julia-package-guidelines and rendered at https://juliapackagefactory.github.io/julia-package-guidelines/.
Placeholders in double curly braces are filled from the query parameters of the rendered page, for example https://juliapackagefactory.github.io/julia-package-guidelines/?package_name=Antique.jl&owner=ohno.

<!-- raw -->
| Placeholder | Query parameter | Example |
|---|---|---|
| `{{package_name}}` | `package_name` | `Antique.jl` |
| `{{module_name}}` | `module_name`, defaults to `package_name` without `.jl` | `Antique` |
| `{{owner}}` | `owner` | `ohno` |
<!-- /raw -->

Badge for a package README:

```md
[![guidelines: JuliaPackageFactory](https://img.shields.io/badge/guidelines-JuliaPackageFactory-blueviolet)](https://juliapackagefactory.github.io/julia-package-guidelines/?package_name={{package_name}}&owner={{owner}})
```

Line for a package's `AGENTS.md`:

```
Follow https://juliapackagefactory.github.io/julia-package-guidelines/README.md with package_name={{package_name}} and owner={{owner}}.
```

Based on [ColPrac](https://github.com/SciML/ColPrac), [Modern Julia Workflows](https://modernjuliaworkflows.org/sharing/), [Pkg.jl: Creating Packages](https://pkgdocs.julialang.org/v1/creating-packages/), and the [Julia workflow tips](https://docs.julialang.org/en/v1/manual/workflow-tips/). [MIT License](LICENSE).
