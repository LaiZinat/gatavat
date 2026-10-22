# Contributing

Issues and pull requests are welcome. The maintainer is Dr. Leonhard Horstmeyer
(mail@leonhardhorstmeyer.com). Please keep the tool dependency-free: Python standard
library only, plus the single `ui.html` file.

## Before you open a pull request

- Run `./llmctl --version` and `./llmctl` to check that the script still loads.
- Test changes against an isolated config so your real setup is untouched:
  `LLMCTL_HOME=$(mktemp -d) ./llmctl ...`
- Update `docs/` and `CHANGELOG.md` when behaviour or settings change.

## Commit messages

Every commit message starts with a keyword, then a short description of what the commit does:

```
<keyword>: <what was done>
<keyword>(<scope>): <what was done>
```

- Write the description in the imperative, lower case, without a trailing period:
  `fix: assign a free port in llmctl add`, not `Fixed port bug.`
- The scope is optional and names the part that changed, e.g. `gateway`, `ui`, `cli`,
  `opencode`, `specs`, `chat`.
- Add a body after a blank line when the *why* isn't obvious.
- Mark breaking changes (config format, defaults, CLI behaviour) with `!` after the keyword and
  explain the migration in the body: `feat(opencode)!: export one provider via the gateway`.

| keyword | use for |
|---|---|
| `init` | The very first commit of the repository. Used exactly once, never again. |
| `feat` | A new feature or user-visible capability |
| `fix` | A bug fix |
| `docs` | Documentation only (README, docs/, CHANGELOG, comments) |
| `refactor` | Code change that neither fixes a bug nor adds a feature |
| `perf` | Performance improvement |
| `test` | Adding or changing tests |
| `style` | Formatting, whitespace, naming; no behaviour change |
| `build` | Packaging, install scripts, dependencies |
| `ci` | CI configuration (e.g. GitHub Actions) |
| `chore` | Maintenance that fits nowhere else (e.g. .gitignore, version bump) |
| `revert` | Reverting an earlier commit; reference its hash in the body |

Examples:

```
feat(gateway): list model state on /v1/models
fix(cli): assign a free port in llmctl add
docs: explain split GGUF paths
chore: release 0.2.0
```

## Releases

Bump `__version__` in `llmctl`, add a section to `CHANGELOG.md`, commit with
`chore: release X.Y.Z`, and tag it `vX.Y.Z`.
