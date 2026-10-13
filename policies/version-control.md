# Version Control Standards

| | |
|---|---|
| **Applies to** | Every repository in the `wayne-pipe-and-supply` GitHub organization |
| **Owner** | IT Manager |
| **Status** | Draft v0.1 |
| **Change process** | Pull request to this repository (see [Changing this policy](#14-changing-this-policy)) |

This document is the IT department's standard for how we use Git and GitHub. It
covers repositories, branches, commits, pull requests, releases, and what must never
be committed. Technology-specific rules live in addenda (see
[Stack addenda](#13-stack-addenda)); an addendum can add requirements but cannot
relax this document.

The words **must**, **must not**, **should**, and **may** carry their usual meaning:
*must* is mandatory, *should* is the default unless you have a reason you can explain
in the PR, *may* is optional.

> **This repository is public.** GitHub requires the organization's `.github`
> repository to be public for its default files to apply to our other repositories.
> Nothing in it may name internal hosts, servers, databases, credentials, customers,
> or vendor-confidential details.

---

## 1. The short version

1. One repository per independently deployed thing. Create it from a template when
   one exists.
2. `main` is the only long-lived branch. It is protected and always deployable.
3. All work happens on a short-lived branch named `<type>/<description>`.
4. Every change reaches `main` through a pull request.
5. PR titles and commit messages follow [Conventional Commits](https://www.conventionalcommits.org/en/v1.0.0/).
6. PRs are squash-merged; the branch is deleted on merge.
7. Releases are SemVer tags (`v1.4.0`) with a `CHANGELOG.md` entry and a GitHub Release.
8. What gets deployed is built from a tag — never from a laptop's working copy.
9. Build output, secrets, credentials, and customer data never go into Git.
10. Every repository has a README a stranger could use to build, deploy, and roll back.

---

## 2. Repositories

### 2.1 One repository per deployable unit

A repository holds **one thing that is built, versioned, and deployed on its own
schedule** — one business rule, one website, one tool. If two things can be released
independently, they belong in separate repositories.

Repository count is not a problem we optimize for. We keep many repositories
navigable with consistent names, descriptions, topics, and READMEs that GitHub search
can find. If the list ever outgrows search, we will add a catalog; until then, don't
combine unrelated things to keep the count down.

### 2.2 Naming

- Format: `wps-<area>-<name>`, all lowercase, words separated by hyphens.
  Examples: `wps-p21-rule-item-id-validator`, `wps-ups-rate-calculator`.
- `<area>` groups related repositories so they sort together: `p21`, `web`, `data`,
  `ops`, and so on. Stack addenda may fix the pattern for their area.
- Names describe what the thing **is**, not who wrote it or when.
- No version numbers in repository names (`-v2`). A rewrite is a major version, not a
  new repository, unless the old one is being archived.
- Existing repositories that don't match are renamed when they are next actively
  worked on. GitHub redirects the old URL; update local remotes with
  `git remote set-url origin <new-url>`.

### 2.3 Visibility

| Visibility | Use for |
|---|---|
| **Internal** (default) | Almost everything. Visible to organization members only. |
| **Private** | Repositories whose content is sensitive even internally — security tooling, HR/finance data handling, anything restricted to specific people. |
| **Public** | Only with IT Manager approval. Currently: this `.github` repository. Abide by all upstream licensing requirements, i.e. when dependencies mandate open sourcing (copyleft or reciprocal licenses). |

### 2.4 Required files

Every repository **must** have:

| File | Purpose |
|---|---|
| `README.md` | What it is, how to build/run/deploy/roll back it. See [§10](#10-documentation). |
| `.gitignore` | Start from the matching [github/gitignore](https://github.com/github/gitignore) template, then add project specifics below it. |
| `.gitattributes` | At minimum `* text=auto` so line endings are normalized across Windows and CI. |
| `.github/CODEOWNERS` | Who reviews changes. GitHub cannot inherit this from the organization. |
| `CHANGELOG.md` | Required for any repository that publishes releases (see [§8](#8-versioning-and-releases)). |

Repositories inherit `CONTRIBUTING.md`, `SECURITY.md`, the pull request template, and
the issue templates from this `.github` repository. A repository **may** override
any of them by adding its own file of the same name.

### 2.5 Repository settings

Set these when the repository is created:

- **Description:** one sentence. It is what search results show.
- **Topics:** at least one for the area (`p21`, `web`, …) and one for the stack
  (`csharp`, `python`, …). Topics are how we filter the repository list.
- **Default branch:** `main`. Existing `master` repositories are renamed when next
  actively worked on.
- **Pull requests:** allow **squash merging only**. Default squash commit message:
  *Pull request title*. Enable *Automatically delete head branches*.
- **Features:** keep Issues on. Turn Wiki off — documentation lives in the
  repository, where it is versioned and reviewed (see [§10](#10-documentation)).

### 2.6 Creating a repository

1. If a template repository exists for the kind of thing you're building (for
   example, `wps-p21-rule-template` for P21 business rules), create from it:
   **New repository → Repository template**.
2. Apply the settings in [§2.5](#25-repository-settings).
3. Fill in the README before the first feature PR. The first PR into a new
   repository is usually just the README, `.gitignore`, and scaffolding.

### 2.7 Retiring a repository

Archive it; don't delete it. Add a line at the top of the README saying why it was
archived and where its replacement lives, then **Settings → Archive this
repository**. Archived repositories are read-only but keep their history.

---

## 3. Branches

### 3.1 `main`

- `main` is the only long-lived branch and is always in a deployable state.
- `main` is protected by an organization ruleset: no deletion, no force-push, and
  changes only through a pull request with one approval (see [§6](#6-admin-bypass)
  for the exception).
- There are **no** long-lived `develop`, `test`, `staging`, or per-environment
  branches. Which version runs in which environment is tracked with tags and
  releases ([§8](#8-versioning-and-releases)), not branches.

### 3.2 Working branches

- Branch from an up-to-date `main` for every change.
- Name: `<type>/<short-description>`, using the commit types from
  [§4.2](#42-types). Include an issue number when one exists.

  ```text
  feat/supplier-prefix-check
  fix/42-blank-supplier-id
  docs/readme-deploy-steps
  chore/gitignore-vs-template
  ```

- Keep branches short-lived — days, not weeks. A branch that lives long drifts from
  `main` and becomes hard to review. Split large work into a sequence of smaller PRs.
- Push your branch at least daily, even unfinished. Your laptop is not a backup.
- Rebasing or force-pushing **your own** working branch is fine; use
  `git push --force-with-lease`, never plain `--force`.
- Branches are deleted automatically when their PR merges. Delete abandoned branches
  yourself.

---

## 4. Commits

### 4.1 Format

Commit messages follow [Conventional Commits 1.0](https://www.conventionalcommits.org/en/v1.0.0/):

```text
<type>(<optional scope>): <description>

<optional body: what changed and why>

<optional footer(s)>
```

- **Description:** imperative mood ("add", not "added" or "adds"), lowercase, no
  trailing period, whole first line ≤ 72 characters.
- **Body:** explain *why*. The diff already shows *what*. Wrap at 72 characters.
- **Scope:** optional; a component or area (`fix(validator): …`).

Because we squash-merge, **the PR title becomes the commit on `main`**, so the PR
title is the message that matters most and must follow this format. Commits on your
working branch should follow it too — good habits there make PRs easier to review —
but they are squashed away.

### 4.2 Types

| Type | Use for |
|---|---|
| `feat` | New behavior a user or another system can notice |
| `fix` | A bug fix |
| `docs` | Documentation only |
| `refactor` | Code change that neither fixes a bug nor adds behavior |
| `perf` | Performance improvement |
| `test` | Adding or fixing tests |
| `build` | Build system, project files, dependencies |
| `ci` | CI workflow changes |
| `chore` | Maintenance that fits nothing above (`.gitignore`, tooling config) |
| `revert` | Reverting an earlier commit |

### 4.3 Breaking changes

Mark a change that requires someone to do something differently — reconfigure,
re-register, migrate data — with `!` after the type/scope **and** a
`BREAKING CHANGE:` footer explaining what they must do:

```text
feat(registration)!: move rule to the order Save event

BREAKING CHANGE: unassign the rule from the field-edit event and assign it
to Save in the rule configuration screen before deploying this version.
```

### 4.4 Examples

| Good | Not good |
|---|---|
| `feat: block item save when ID lacks supplier prefix` | `Added new project` |
| `fix: trim fixed-width route value before comparing` | `Final version` |
| `chore: replace .gitignore with Visual Studio template` | `Update .gitignore` |
| `docs: add deploy and rollback steps to README` | `changes` |
| `build: reference P21.Extensions as a package` | `fixed build` (also: three commits with the same message) |

### 4.5 What a commit should contain

- **One logical change.** If the description needs "and", consider two commits.
- **Source, not output.** Never commit `bin/`, `obj/`, `.vs/`, compiled DLLs/EXEs,
  logs, or caches. If `git status` shows build output, fix `.gitignore` first.
- **Review before committing.** Run `git status` and `git diff --staged` every time.
  `git add -A` without looking is how build output and secrets get committed.

### 4.6 Identity

- Commit with your **company email address** or your GitHub `noreply` address, never
  a personal address:

  ```bash
  git config --global user.name  "First Last"
  git config --global user.email "you@waynepipe.com"
  ```

- Use one GitHub account for company work, a member of this organization, with
  two-factor authentication on.
- AI-assisted commits may carry `Co-Authored-By:` trailers. The human who commits is
  still responsible for the content.

---

## 5. Pull requests

### 5.1 Everything goes through a PR

Every change to `main` arrives through a pull request, including documentation and
one-line fixes. The PR is the record of what changed, why, who reviewed it, and how
it was tested.

### 5.2 Opening a PR

- **Title:** a Conventional Commit line ([§4](#4-commits)) — it becomes the commit on
  `main`. A CI check enforces the format where the repository has it enabled.
- **Description:** fill in the template. Say what changed, why, and how you tested
  it. Link the issue (`Closes #42`).
- **Size:** aim for something a reviewer can review properly in under 30 minutes.
  Large PRs get rubber-stamped; small ones get reviewed.
- **Draft PRs:** open one early to share work in progress or ask for direction.
  Mark it *Ready for review* when it is.

### 5.3 Review

- **One human approval** from someone other than the last person to push is
  required (enforced by the ruleset). New pushes dismiss earlier approvals.
- **Automated review:** where `claude[bot]` review is enabled, the author addresses
  its findings (fix, or reply with why not) before asking for human review.
  Automated review supplements human review; it does not replace it, except as
  described in [§6](#6-admin-bypass).
- **Reviewers** check that the change does what the description says, that it is
  tested, that documentation and `CHANGELOG.md` are updated, and that nothing in
  [§9](#9-what-never-goes-in-git) is being committed. Review the code, not the person.
- **Authors** resolve every comment, by changing the code or replying, before merging.

### 5.4 Merging

- **Squash and merge** only. Check that the final commit title is still a correct
  Conventional Commit line.
- The author merges once approved, unless agreed otherwise.
- The branch is deleted automatically.

### 5.5 Reverting

To undo something on `main`, use GitHub's **Revert** button on the merged PR (or
`git revert` on a branch) and merge the revert PR. Never rewrite `main`'s history.

---

## 6. Admin bypass

Organization and repository administrators can bypass the pull request requirement on
`main`. The bypass exists for two situations:

1. **Solo admin work.** An admin working alone opens a PR as usual, lets the
   automated review (`claude[bot]`) run, addresses its findings, and bypass-merges
   once the review reports no blocking issues. The PR still exists as the record.
2. **Urgent changes.** When waiting for review would cause harm — a production rule
   blocking order entry, a broken site — an admin may push directly to `main`. The
   commit must still follow [§4](#4-commits), and the admin should open a follow-up
   PR or issue describing what was done so it gets a second look.

The bypass is **not** for skipping review of a non-admin's work, and it does not
relax anything else in this policy. Bypasses are recorded in the ruleset's insights
log.

---

## 7. Keeping in sync

- Before starting work: `git switch main && git pull`.
- Before opening a PR, bring your branch up to date with `main`: either
  `git rebase main` (then `--force-with-lease`) or GitHub's *Update branch* button.
  Either is fine; the squash merge leaves `main` linear regardless.
- Resolve conflicts on your branch, never by pushing to `main`.

---

## 8. Versioning and releases

Any repository that produces a deployable artifact (a DLL, an executable, a package)
**must** publish versioned releases. Continuously deployed repositories (for example,
websites that deploy on every merge to `main`) **may** skip tagging; for them, the
merge to `main` is the release.

### 8.1 Semantic Versioning

Versions follow [SemVer 2.0](https://semver.org/): `MAJOR.MINOR.PATCH`.

- **MAJOR:** someone must do something differently to adopt it — reconfiguration,
  re-registration, data migration (anything marked `!` / `BREAKING CHANGE`).
- **MINOR:** new behavior, backward compatible.
- **PATCH:** bug fixes, no intended behavior change.
- Pre-releases for testing environments use a suffix: `v1.4.0-rc.1`.
- `0.x` versions are for work that has never reached production.

Stack addenda define what "breaking" means for their kind of artifact.

### 8.2 Tags

- Tags are annotated and prefixed with `v`: `git tag -a v1.4.0 -m "v1.4.0"`.
- Tags are created on `main` (or via a GitHub Release, which tags for you) and are
  never moved or deleted once pushed.
- The version inside the code (assembly version, `pyproject.toml`, `package.json`)
  must match the tag. Bump it in the same PR that updates `CHANGELOG.md`.

### 8.3 `CHANGELOG.md`

Format: [Keep a Changelog 1.1](https://keepachangelog.com/en/1.1.0/).

- Keep an `## [Unreleased]` section at the top. Each PR that changes behavior adds
  a line under it (`Added`, `Changed`, `Deprecated`, `Removed`, `Fixed`, `Security`).
- Releasing means renaming `[Unreleased]` to the version and date in a
  `chore(release): v1.4.0` PR.
- Write entries for the person deploying or supporting the software, not for
  developers. "Item save is blocked when the item ID contains spaces", not
  "updated regex".

### 8.4 GitHub Releases and artifacts

- Every version gets a GitHub Release whose notes are its `CHANGELOG.md` entry.
- Build artifacts are **attached to the Release**, not committed to the repository.
- Artifacts are built from the tagged commit in a clean clone or in CI, in Release
  configuration — never from a working copy with uncommitted changes.
- **Promote the artifact, don't rebuild it.** The exact file that passed testing in
  one environment is the one deployed to the next.
- At any time, we must be able to answer "what version is running where?" from the
  repository: the stack addendum says where deployments are recorded.

---

## 9. What never goes in Git

Git history is permanent and copied to every clone. Deleting a file in a later
commit does **not** remove it from history.

**Never commit:**

- Passwords, API keys, tokens, consumer keys, certificates, private keys, or
  connection strings that contain credentials.
- Customer data or personal data: real orders, invoices, captured data streams,
  exports, screenshots of customer records. Use synthetic or redacted test data.
- Build output: `bin/`, `obj/`, `dist/`, compiled binaries, installers, logs, caches.
- IDE and machine state: `.vs/`, `.idea/`, `*.user`, `*.suo`, personal editor
  settings, absolute paths to your own machine.
- Vendor or licensed binaries, unless a stack addendum explicitly allows a pinned
  copy (for example, a vendored package that cannot be restored from a public feed).
- Large files (over about 5 MB). Attach them to a Release or discuss Git LFS with
  the IT Manager.

Configuration that differs per environment or contains secrets goes in environment
variables, a secrets manager, or an untracked local file with a committed
`*.example` showing its shape.

**If something sensitive is committed:**

1. **Stop.** Don't try to hide it with another commit, and don't open a public issue.
2. **Tell the IT Manager directly**, right away. See [`SECURITY.md`](../SECURITY.md).
3. **Rotate first.** Any exposed credential is treated as compromised and rotated
   before anything else, even if the repository is internal.
4. **Then purge history** with `git filter-repo`, coordinated with the IT Manager
   because it rewrites history for everyone with a clone.

---

## 10. Documentation

### 10.1 README

The README is the front door. Someone who has never seen the repository must be able
to answer from it alone:

- **What** is this, and **why** does it exist? (business purpose in plain language)
- **Status:** in development, in testing, in production — and where.
- **How to build** it from a fresh clone, including prerequisites.
- **How to run/test** it.
- **How to deploy** it, and **how to roll it back**.
- **Configuration** it needs, and where that lives (never the secret values).
- **Related** repositories and documents.

Stack addenda and templates specify the exact sections for their kind of repository.

### 10.2 Other documentation

- Longer documents live in `docs/` inside the repository, so they are versioned and
  reviewed with the code. We don't use GitHub wikis.
- Design documents and plans are dated: `docs/YYYY-MM-DD-<topic>.md`. Write the design
  before the code for anything non-trivial, and link it from the PR.
- Comments in code explain *why*. Deployment dates, version history, and change logs
  do not belong in code comments — that is what tags, Releases, and `CHANGELOG.md` are
  for.

---

## 11. Bringing existing repositories into line

- **New repositories** follow this policy from their first commit.
- **Existing repositories** are brought into line when they are next actively worked
  on: required files, settings, naming, default branch. A `chore:` PR doing just that
  is the usual first step.
- **Restructuring** (splitting a repository, removing committed build output from
  history) is planned with the IT Manager, because rewriting history affects
  everyone who has a clone.

---

## 12. Quick reference

```bash
# start
git switch main && git pull
git switch -c feat/short-description

# work
git status
git add <files>
git diff --staged
git commit -m "feat: describe the change in the imperative"

# publish and open a PR
git push -u origin feat/short-description
gh pr create --fill          # or use the link git prints

# update with main
git fetch origin
git rebase origin/main
git push --force-with-lease

# after merge
git switch main && git pull
git branch -d feat/short-description
```

---

## 13. Stack addenda

| Addendum | Covers |
|---|---|
| [P21 business rules](p21-business-rules.md) | Prophet 21 business rule DLLs |

Add a new addendum when a stack has requirements this document doesn't cover.

---

## 14. Changing this policy

This policy follows its own rules: propose a change with a PR to this repository,
with a Conventional Commit title (`docs(policy): …`). The IT Manager approves
changes. Record notable changes below.

| Version | Date | Change |
|---|---|---|
| 0.1 | 2026-10-13 | Initial draft. |
