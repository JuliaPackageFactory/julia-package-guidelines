# Julia Package Development Guide

**Package:** `{{package_name}}` · **Repository:** https://github.com/{{owner}}/{{package_name}}

This guide describes how to develop, test, document, and release `{{package_name}}`.
It is written for humans and AI agents alike: every step is a command that runs from
the repository root and works the same on Linux, macOS, and Windows.
It assumes [Julia](https://julialang.org/install/) 1.12 or later and [Git](https://git-scm.com/).

The guide follows four principles.

- **Reproducible.** Commands pass `--startup-file=no` and an explicit `--project`, so they behave the same on every machine and in CI.
- **Clean.** Development tools (Revise, Runic) live in named shared environments such as `@revise`, never in `Project.toml` or the default environment. `Project.toml` lists only what the package needs at runtime; `test/` and `docs/` keep their own `Project.toml` inside a Pkg workspace.
- **Checked before shared.** Format, tests, and docs pass locally before a push. CI repeats the same checks.
- **Small steps.** One pull request per purpose, following [ColPrac](https://github.com/SciML/ColPrac).

## 1. Install tools (once per machine)

Install [Revise.jl](https://github.com/timholy/Revise.jl) (reloads code without restarting Julia) and [Runic.jl](https://github.com/fredrikekre/Runic.jl) (the formatter) into their own shared environments.

```sh
julia --startup-file=no --project=@revise -e 'using Pkg; Pkg.add("Revise")'
julia --startup-file=no --project=@runic  -e 'using Pkg; Pkg.add("Runic")'
```

These environments live in `~/.julia/environments/` and are shared by all packages you work on.
Update them occasionally with the same commands, replacing `Pkg.add("...")` by `Pkg.update()`.

## 2. Set up the repository (once per package)

Maintainers clone the repository directly.
Contributors first [fork](https://github.com/{{owner}}/{{package_name}}/fork) it on GitHub, clone the fork, and add the original repository as `upstream`.

```sh
git clone https://github.com/{{owner}}/{{package_name}}.git
cd {{package_name}}
julia --startup-file=no --project=. -e 'using Pkg; Pkg.instantiate()'
```

The last command downloads the dependencies declared in `Project.toml` and writes `Manifest.toml`, which is ignored by Git.
Contributors clone their fork instead (replace `{{owner}}` by their own account) and then run `git remote add upstream https://github.com/{{owner}}/{{package_name}}.git`.

The repository is laid out as follows.

```
{{package_name}}/
├── Project.toml           # name, uuid, version, [deps], [compat], [workspace]
├── src/{{module_name}}.jl # the module
├── test/                  # test/Project.toml and runtests.jl
├── docs/                  # docs/Project.toml, make.jl, and src/*.md
├── CHANGELOG.md           # user-visible changes, Keep a Changelog format
├── CITATION.cff           # how to cite; version and date must match releases
├── README.md
├── LICENSE
└── .github/workflows/     # CI, formatting check, TagBot, dependency updates
```

## 3. Develop (every change)

### 3.1 Create a branch

Open an [issue](https://github.com/{{owner}}/{{package_name}}/issues) first if the change is significant, then branch from `main`.
Name the branch after the issue (`issue/123`) or the topic (`feature/fast-solver`).

```sh
git switch main
git pull
git switch -c issue/123
```

### 3.2 Start a Julia session with Revise

```sh
julia --project=. -i -e 'insert!(LOAD_PATH, 2, "@revise"); using Revise; using {{module_name}}'
```

`insert!(LOAD_PATH, 2, "@revise")` makes the `@revise` environment visible to this session only, right after the package's own environment.
Edit files under `src/`; Revise applies the changes to the running session.
Restart Julia only when you change a `struct` definition or `Project.toml`.

### 3.3 Edit the code

- **Docstrings.** Every exported or `public` name gets a docstring. See [Documenter: Adding docstrings](https://documenter.juliadocs.org/stable/man/guide/#Adding-Some-Docstrings) for the format.
- **Tests.** Add or update tests under `test/` for every behavior you add or fix.
- **Changelog.** Describe user-visible changes under `## [Unreleased]` in `CHANGELOG.md`.
- **Dependencies.** Add a dependency with Pkg. On Julia 1.12 this also writes a `[compat]` entry with the installed version as the lower bound; widen it if older versions are fine. The General registry requires compat bounds for all dependencies.

  ```sh
  julia --startup-file=no --project=. -e 'using Pkg; Pkg.add("SomePackage")'
  julia --startup-file=no --project=. -e 'using Pkg; Pkg.compat("SomePackage", "1.2")'
  ```

  Test-only and docs-only dependencies go into `test/Project.toml` and `docs/Project.toml` instead (`--project=test`, `--project=docs`).

### 3.4 Format

Runic formats all `.jl` files in place. The CI formatting check fails on unformatted code, so run this before every commit.

```sh
julia --startup-file=no --project=@runic -m Runic --inplace .
```

To check without modifying files, replace `--inplace` by `--check --diff`.

### 3.5 Test

```sh
julia --startup-file=no --project=. -e 'using Pkg; Pkg.test()'
```

This runs `test/runtests.jl` in a fresh process with the test dependencies.
Quality checks such as [Aqua.jl](https://github.com/JuliaTesting/Aqua.jl) and [JET.jl](https://github.com/aviatesk/JET.jl), if the package uses them, run as part of the same test suite.

For a faster loop while iterating, start a Revise session in the test environment and re-include the tests after each edit:

```sh
julia --project=test -i -e 'insert!(LOAD_PATH, 2, "@revise"); using Revise; using {{module_name}}'
```

```julia
julia> include("test/runtests.jl")
```

### 3.6 Build the documentation

```sh
julia --startup-file=no --project=docs -e 'using Pkg; Pkg.instantiate()'
julia --startup-file=no --project=docs docs/make.jl
```

Open `docs/build/index.html` in a browser. The build also runs the doctests in docstrings, so it fails on errors in `jldoctest` blocks.
The first command is needed only once, or after `docs/Project.toml` changes.

### 3.7 Commit and open a pull request

Commit once format, tests, and docs succeed. Do not commit `Manifest.toml`, `docs/build/`, or coverage files.

```sh
git add src test docs CHANGELOG.md
git commit -m "Add fast solver (#123)"
git push -u origin issue/123
```

Then open a pull request on GitHub (contributors: from the fork to `{{owner}}/{{package_name}}`).
In the description, state what changed and why, link the issue, and flag breaking changes.

## 4. Review and merge

The project follows [ColPrac](https://github.com/SciML/ColPrac). In short:

- A pull request does one thing. Keep refactors, style-only changes, and features in separate pull requests.
- Every pull request needs passing CI and one approval from someone other than the author.
- Reviewers phrase comments as questions and use GitHub's suggestion feature for small fixes.
- Do not squash or force-push while a review is in progress; reviewers need to see the new commits.
- Changes reach `main` only through pull requests.

## 5. Release (maintainers)

### 5.1 Choose the version

`{{package_name}}` follows [Semantic Versioning](https://semver.org/) as interpreted by Pkg and ColPrac.
The public API is everything exported, marked `public`, or documented as public.

| Change | Version ≥ 1.0.0 | Version < 1.0.0 |
|---|---|---|
| Breaking change to the public API or to a documented example | major (`1.2.3 → 2.0.0`) | minor (`0.2.3 → 0.3.0`) |
| New feature, new export, dropped support for an old Julia version | minor (`1.2.3 → 1.3.0`) | patch (`0.2.3 → 0.2.4`) |
| Bug fix, docs, internal change, dependency bump | patch (`1.2.3 → 1.2.4`) | patch (`0.2.3 → 0.2.4`) |

Bug fixes are released as soon as they are merged. Breaking releases are rare and deliberate.
When breaking work is merged before it is released, set `version = "2.0.0-DEV"` in `Project.toml` and drop the `-DEV` suffix in the release pull request.

### 5.2 Prepare the release pull request

Update three files in one pull request titled `Release vX.Y.Z`:

1. `Project.toml`: `version = "X.Y.Z"`.
2. `CHANGELOG.md`: rename `## [Unreleased]` to `## [X.Y.Z] - YYYY-MM-DD` and start a new empty `## [Unreleased]` section above it.
3. `CITATION.cff`: `version: "X.Y.Z"` and `date-released: "YYYY-MM-DD"`. Check the author list, and add `orcid` or DOI `identifiers` only when they exist.

Merge the pull request after CI passes.

### 5.3 Register

Comment the following on the merge commit (or on the release pull request):

```
@JuliaRegistrator register
```

[Registrator](https://github.com/JuliaRegistries/Registrator.jl) opens a pull request against the [General](https://github.com/JuliaRegistries/General) registry.
It merges automatically after 15 minutes for new versions, or after 3 days for a brand-new package, provided the automatic checks pass.
[TagBot](https://github.com/JuliaRegistries/TagBot) then creates the Git tag and the GitHub release, and the documentation workflow deploys the `stable` docs.

Before the first registration:

- Install the [Registrator GitHub App](https://github.com/apps/juliateam-registrator) on the repository.
- Check the [package naming guidelines](https://pkgdocs.julialang.org/v1/creating-packages/#Package-naming-guidelines): ASCII letters and digits only, at least five characters, and no name that is easily confused with an existing package.
- Make sure every dependency, including `julia`, has a `[compat]` entry.

## 6. Maintain

- **Dependency updates.** Dependabot or CompatHelper opens pull requests that widen `[compat]` entries. Merge them when CI passes; they are patch-level changes.
- **Tool updates.** Refresh the shared environments now and then: `julia --startup-file=no --project=@runic -e 'using Pkg; Pkg.update()'`, and likewise for `@revise`.
- **Dropping a Julia version.** Raise `julia = "..."` in `[compat]` and release a minor (≥ 1.0) or patch (< 1.0) version. Give a reason in the changelog.
- **Accidental breaking release.** Revert the offending change, release a patch, and either reintroduce the change in a non-breaking way or plan a proper major release. A badly broken version can additionally be [yanked](https://github.com/JuliaRegistries/General#how-do-i-yank-a-version) from the registry.
- **Issues.** Triage new issues within a few days: reproduce, label, and link to the pull request that fixes them.

## Cheat sheet

```sh
# once per machine
julia --startup-file=no --project=@revise -e 'using Pkg; Pkg.add("Revise")'
julia --startup-file=no --project=@runic  -e 'using Pkg; Pkg.add("Runic")'

# once per package
git clone https://github.com/{{owner}}/{{package_name}}.git
cd {{package_name}}
julia --startup-file=no --project=. -e 'using Pkg; Pkg.instantiate()'

# every change
git switch -c issue/123
julia --project=. -i -e 'insert!(LOAD_PATH, 2, "@revise"); using Revise; using {{module_name}}'
julia --startup-file=no --project=@runic -m Runic --inplace .
julia --startup-file=no --project=. -e 'using Pkg; Pkg.test()'
julia --startup-file=no --project=docs -e 'using Pkg; Pkg.instantiate()'
julia --startup-file=no --project=docs docs/make.jl
git add src test docs CHANGELOG.md && git commit -m "..." && git push -u origin issue/123

# release (maintainers): bump Project.toml, CHANGELOG.md, CITATION.cff, merge,
# then comment "@JuliaRegistrator register" on the merge commit
```

## For AI agents

When you work on `{{package_name}}`, follow this guide and these rules.

1. Run commands from the repository root exactly as written above. Do not use the Pkg REPL mode; use `julia -e '...'`.
2. Never add Revise, Runic, or other development tools to `Project.toml`. They belong in the `@revise` and `@runic` environments.
3. Before every commit, run format ([3.4](#34-format)), tests ([3.5](#35-test)), and the docs build ([3.6](#36-build-the-documentation)), and fix what fails.
4. Keep code, tests, docstrings, and `CHANGELOG.md` consistent in the same change.
5. Do not commit `Manifest.toml`, `docs/build/`, `*.cov`, or other generated files.
6. Do not change `version` in `Project.toml`, `CITATION.cff`, or release headings in `CHANGELOG.md` unless the task is explicitly a release.
7. Do not push to `main`; work on a branch and open a pull request.

To point an agent at this guide from a package, add a line like this to the package's `AGENTS.md`:

```
Follow https://juliapackagefactory.github.io/Guidelines/README.md with package_name={{package_name}} and owner={{owner}}.
```

## About this guide

This document is maintained at https://github.com/JuliaPackageFactory/Guidelines and rendered at https://juliapackagefactory.github.io/Guidelines/.
It is a template: placeholders written as a name in double curly braces are replaced by query parameters of the rendered page.

<!-- raw -->
| Placeholder | Query parameter | Example | Meaning |
|---|---|---|---|
| `{{package_name}}` | `package_name` | `Antique.jl` | Repository and directory name, with `.jl` |
| `{{module_name}}` | `module_name` (optional) | `Antique` | Module name; defaults to `package_name` without `.jl` |
| `{{owner}}` | `owner` | `ohno` | GitHub user or organization |
<!-- /raw -->

For example, https://juliapackagefactory.github.io/Guidelines/?package_name=Antique.jl&owner=ohno renders the guide for Antique.jl.
Without JavaScript, or when reading the Markdown source, substitute the placeholders yourself.
Packages generated by [PkgFactory](https://github.com/JuliaPackageFactory/PkgFactory.ts) map `PKG` to `module_name` and `OWNER` to `owner`.

To link to this guide from a package README:

```md
[![guidelines: JuliaPackageFactory](https://img.shields.io/badge/guidelines-JuliaPackageFactory-blueviolet)](https://juliapackagefactory.github.io/Guidelines/?package_name={{package_name}}&owner={{owner}})
```

This guide builds on [ColPrac](https://github.com/SciML/ColPrac), [Modern Julia Workflows](https://modernjuliaworkflows.org/sharing/), [Pkg.jl: Creating Packages](https://pkgdocs.julialang.org/v1/creating-packages/), and the [Julia workflow tips](https://docs.julialang.org/en/v1/manual/workflow-tips/).
Improvements are welcome as issues and pull requests. Licensed under the [MIT License](LICENSE).
