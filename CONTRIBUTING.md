# Contributing

This is the default contributing guide for every repository in the
`wayne-pipe-and-supply` organization. A repository may add its own `CONTRIBUTING.md`
with project-specific setup; this guide still applies.

The full rules are in the [Version Control Standards](policies/version-control.md).
This page is the day-to-day workflow.

## Before you start

- Your Git identity uses your company email (or GitHub `noreply`), not a personal
  address:

  ```bash
  git config --global user.name  "First Last"
  git config --global user.email "you@waynepipe.com"
  ```

- Read the repository's `README.md` for build and test steps.
- For anything bigger than a small fix, open or find an issue first so the work is
  agreed before it's built.

## The workflow

```bash
# 1. Start from an up-to-date main
git switch main
git pull

# 2. Create a branch: <type>/<short-description>
git switch -c fix/42-blank-supplier-id

# 3. Make the change. Commit in small, logical steps.
git status                       # check what changed — no bin/, obj/, secrets
git add <files>
git diff --staged                # read what you are about to commit
git commit -m "fix: reject blank supplier ID before prefix check"

# 4. Push and open a pull request
git push -u origin fix/42-blank-supplier-id
gh pr create --fill              # or open the link git prints
```

5. **Fill in the PR template.** Make the PR title a Conventional Commit line — it
   becomes the commit on `main`.
6. **Address the automated review** (if enabled), then request a human review.
7. **Respond to every review comment** — change the code or explain why not.
8. **Squash and merge** when approved. The branch is deleted automatically.
9. Clean up locally:

   ```bash
   git switch main
   git pull
   git branch -d fix/42-blank-supplier-id
   ```

## Commit and PR titles

[Conventional Commits](https://www.conventionalcommits.org/en/v1.0.0/):
`<type>(<optional scope>): <imperative description>`

| Type | For |
|---|---|
| `feat` | New behavior |
| `fix` | Bug fix |
| `docs` | Documentation only |
| `refactor` | Restructure, no behavior change |
| `test` | Tests |
| `build` | Project files, dependencies |
| `ci` | Workflows |
| `chore` | Maintenance (`.gitignore`, tooling) |

Add `!` and a `BREAKING CHANGE:` footer when someone has to do something differently
to adopt the change.

## Never commit

Passwords, keys, tokens, connection strings with credentials, customer or personal
data, build output (`bin/`, `obj/`, DLLs, EXEs), IDE state (`.vs/`, `*.user`), or log
files. If you commit one by accident, stop and follow [`SECURITY.md`](SECURITY.md).

## Questions

Ask the IT Manager. If the standards are unclear or wrong, propose a change with a
PR to this repository.
