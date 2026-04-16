# Git Workflow

## Commit Message Format

Follow [Conventional Commits](https://www.conventionalcommits.org/):
```
<type>(<scope>): <description>

[optional body]

[optional footer(s)]
```

**Types**: `feat`, `fix`, `docs`, `style`, `refactor`, `test`, `chore`, `perf`, `ci`, `build`, `revert`

**Scopes**: Use the affected module or feature area (e.g., `auth`, `api`, `db`, `ui`)

**Rules**:
- Subject line: imperative mood, no period, max 72 chars
- Breaking changes: add `!` after type (e.g., `feat!:`) or `BREAKING CHANGE:` in footer
- Reference issues: use `Closes #<number>` or `Fixes #<number>` in footer

Note: Attribution disabled globally via ~/.claude/settings.json.

## Pull Request Workflow

When creating PRs:
1. Analyze full commit history (not just latest commit)
2. Use `git diff [base-branch]...HEAD` to see all changes
3. Draft comprehensive PR summary
4. Include test plan with TODOs
5. Push with `-u` flag if new branch

> For the full development process (planning, TDD, code review) before git operations,
> see [development-workflow.md](./development-workflow.md).
