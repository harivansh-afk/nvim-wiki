# Neovim contribution and maintenance workflow

> Sources: Neovim project docs, Unknown
> Raw: [CONTRIBUTING](../../raw/neovim/CONTRIBUTING.md); [MAINTAIN](../../raw/neovim/MAINTAIN.md); [Developer test reference](../../raw/neovim/runtime/doc/dev_test.txt)
> Updated: 2026-04-23

## Overview

Neovim’s contributor docs combine day-to-day engineering expectations with a maintainer-oriented release and deprecation policy. The common themes are practical debugging, mandatory test coverage, concise communication, and a maintenance strategy that prefers written decisions and automation over informal process.

## Contributor workflow

- New contributors are directed to the developer quickstart and low-risk issues before tackling larger changes.
- Bug reports should start with the FAQ, existing issues, the latest version, and a clean reproduction using `nvim --clean` or a minimal config.
- For regressions and crashes, the docs strongly encourage config bisection, source bisection, stacktraces, sanitizer output, logs, and build-system diagnostics.
- New functionality is generally expected to land in Lua rather than C when practical.
- Contributors are encouraged to use `ninja`, `ccache`, local linting, and targeted test runs to shorten iteration loops.

## Pull request and commit expectations

- Contributors should open draft PRs while work is still in progress and switch to ready-for-review only when the change is actually reviewable.
- PRs are expected to include test coverage, avoid unrelated cosmetic edits, use feature branches, and follow a rebase workflow.
- Commit messages follow a conventional-commit style `type(scope): subject`, with a concise body split into explicit `Problem:` and `Solution:` sections.
- Breaking changes are marked with `!` in the subject and a `BREAKING CHANGE` footer.
- CI is a hard gate: compiler warnings, failing tests, formatting issues, and analyzer failures all matter.
- AI-assisted work is allowed, but the human contributor is expected to review it carefully and remove verbosity, duplication, and weak explanations before requesting review.

## Maintenance and release policy

- Maintainers are encouraged to decide by cost-benefit, write decisions down, embrace constraints, and use automation to solve recurring process problems.
- `master` is treated as the early channel. Maintenance releases are cut from `release-x.y` branches after fixes land on `master` and are cherry-picked back.
- Release automation includes scripts and CI workflows for release asset updates and backports.
- Feature removal follows an explicit cycle of soft deprecation, hard deprecation, then removal across later releases, with documentation and annotations introduced early.
- Dependency management is divided into bundled, vendored, and operational dependencies, each with different update expectations.
- Some inherited legacy modules, notably `regexp.c` and `indent_c.c`, are considered frozen enough that significant restructuring is discouraged until ownership changes.

## Operational conventions

- GitHub Actions is the main CI and automation surface.
- Special labels trigger actions such as backports and extra CI variants.
- Maintainers prefer newer runner tags for low-risk utility jobs, explicit latest versions for core test jobs, and older stable runners for release production where compatibility matters most.

## See Also

- [Neovim project goals and layout](project-goals-and-layout.md)
- [Neovim build, install, and test workflow](build-install-and-test-workflow.md)
