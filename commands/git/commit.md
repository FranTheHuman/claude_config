# /git:commit — Analyze diff and suggest commit messages (English)

Run `git diff --staged` (or `git diff HEAD` if nothing is staged),
analyze the changes, and suggest 3 commit message options following
Conventional Commits format.

## Instructions

1. Run `git diff --staged` to get staged changes.
2. If the output is empty, run `git diff HEAD` to get unstaged changes.
3. If both are empty, run `git status` and report what is going on.
4. Read the diff carefully — understand WHAT changed and WHY it matters.
5. Generate exactly 3 commit message options ranked by specificity.
6. Output the analysis and options in the structure below.

## Output Structure

### What changed
2–3 sentences describing the actual impact of the diff — not just the files touched,
but what behavior, contract, or structure changed. Flag anything that looks like
a breaking change or an unintended side effect.

### Commit message options

**Option 1 — specific** (recommended)
```
<type>(<scope>): <concise description of the actual change>

[optional body: one line explaining WHY if not obvious from the title]
```

**Option 2 — broader scope**
```
<type>(<scope>): <slightly higher-level description>
```

**Option 3 — minimal**
```
<type>: <short description without scope>
```

### Flags (if any)
- ⚠️ Breaking change detected → suggest `!` after type or `BREAKING CHANGE:` footer
- ⚠️ Multiple concerns in one diff → suggest splitting into separate commits
- ⚠️ Debug code left → `console.log`, `System.out.println`, `logger.debug` found

## Conventional Commits type reference

| Type | When to use |
|------|------------|
| `feat` | New feature or capability |
| `fix` | Bug fix |
| `refactor` | Code change with no behavior change |
| `perf` | Performance improvement |
| `test` | Adding or fixing tests |
| `docs` | Documentation only |
| `chore` | Build, deps, config (no production code) |
| `style` | Formatting, whitespace (no logic change) |
| `ci` | CI/CD pipeline changes |

## Examples of good commit messages

```
feat(orders): add retry logic for failed payment processing
fix(auth): prevent session token refresh on expired JWT
refactor(users): extract email validation to shared utility
perf(catalog): replace N+1 query with single JOIN in product listing
test(payments): add integration test for duplicate charge prevention
chore(deps): upgrade Quarkus to 3.8.1
```

## Usage

Run from the root of the repo where the changes are staged or unstaged.

`/git:commit` — no arguments needed, reads the diff automatically.
