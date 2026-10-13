# Addendum: Prophet 21 Business Rules

| | |
|---|---|
| **Extends** | [Version Control Standards](version-control.md) |
| **Applies to** | Repositories containing Prophet 21 (P21) business rule DLLs |
| **Owner** | IT Manager |
| **Status** | Draft v0.1 |

Everything in the core policy applies. This addendum adds what is specific to P21
business rules: one compiled DLL per rule, registered in P21's Business Rule
Organizer, triggered by a P21 window event, and promoted through the Play
environments before production.

> This file lives in a public repository. Keep server names, database names, URLs,
> customer IDs, and credentials out of it — and out of any example you add.

---

## 1. One rule, one repository

- Each business rule gets its own repository, created from
  **`wps-p21-rule-template`**.
- Name: `wps-p21-rule-<what-it-does>`, e.g. `wps-p21-rule-item-id-validator`,
  `wps-p21-rule-will-call-by-route`. Name the behavior, not the screen or the
  developer.
- Topics: `p21`, `p21-business-rule`, `csharp`.
- Visibility: **internal**. Never public — rule repositories contain Epicor's
  licensed `P21.Extensions` package (see §3).
- A rule and the plain classes it uses live together in the same repository. If two
  rules genuinely need the same code, raise it with the IT Manager before creating a
  shared library — every DLL a rule depends on has to be deployed and versioned
  alongside it.

## 2. Project standards

- **Target:** `netstandard2.0`, SDK-style project, built with `dotnet build`.
- **P21 SDK:** the `P21.Extensions` NuGet package at the version pinned in the
  template, restored through the repository's `nuget.config`.
- **Assembly name:** `WPS.P21.<RuleName>`. The rule class name is the
  `<RuleName>` in PascalCase.
- **Builds from a clean clone.** `git clone` → `dotnet build -c Release` must
  succeed on any team machine and in CI. No `HintPath` pointing outside the
  repository (Desktop, Downloads, another repository).
- **Keep logic testable.** The class that inherits `Rule` is a thin adapter: it
  reads fields from P21, calls plain C# classes that make the decision, and writes
  the result back. The decision logic has unit tests in `tests/`, which run in CI
  without P21.
- **Deploy Release builds only.** Debug builds never leave a developer's machine.

## 3. What may be committed

The core policy's [§9](version-control.md#9-what-never-goes-in-git) applies, with one
exception:

- **Allowed:** the pinned Epicor `.nupkg` files in `packages/`: `P21.Extensions` and
  its dependency `P21.Popup.Model`. They are licensed packages that aren't on
  nuget.org, so the repository vendors pinned copies to stay buildable from a clean
  clone and in CI. They are dependencies, not build output.
- **Never:** `bin/`, `obj/`, loose `P21.Extensions.dll` or other DLLs, `.pdb` files,
  rule log output, captured form data streams or field dumps containing customer
  data, decompiled Epicor source, credentials helpers.

## 4. README

The template's README has these sections. All of them must be filled in before the
rule is deployed to Play2:

| Section | Contents |
|---|---|
| Purpose | The business problem, in words a non-developer understands |
| Status | Which environments it is in, and the version in each |
| Registration | Rule type (Validator / Converter / On Event / …), window, triggering event or field, Single/Multi-row, Synchronous/Asynchronous, the rule class name |
| Required fields | Every field the rule reads or writes, with the alias that must be selected in Field Selector |
| Behavior | What it does when the rule passes, when it blocks, and when something unexpected happens (fails open or fails closed, and why) |
| Configuration | Anything that may change without a code change (and where it lives), or an explicit "none" |
| Build and test | Commands from a clean clone |
| Deploy | Step by step, including where the DLL goes and the Organizer configuration |
| Rollback | Step by step (see §7) |
| Related | Other rules touching the same window or data |

Rule-specific header comments in the `.cs` file should describe *how the code works*.
Deployment dates and version history go in `CHANGELOG.md`, not in code comments.

## 5. Versioning

Rules follow core [§8](version-control.md#8-versioning-and-releases). For a business
rule, the SemVer levels mean:

| Level | When |
|---|---|
| **MAJOR** | Someone has to change the rule's setup in P21: event, window, rule type, selected fields or aliases, rule name, or a new required data setup |
| **MINOR** | Behavior changes with the same registration — a new check, a changed condition |
| **PATCH** | Bug fix or message wording, no intended behavior change |

- The version lives in `Directory.Build.props` (`<Version>`) and is bumped in the
  release PR together with `CHANGELOG.md`.
- **The rule name never carries a version.** P21 links a rule by the name it reports
  (`[RuleDescription]` name / `GetName()`, stored in `business_rule.rule_name`), not by
  DLL file. Renaming it (`MyRuleV8`) orphans every existing link and forces the rule to
  be re-linked and its fields re-selected. Git tags are the version history.

### 5.1 Build provenance

Every rule DLL, on any machine or server, must identify the commit it was built from.
The template's `Directory.Build.targets` does this automatically; don't remove it.

| Where | What it shows |
|---|---|
| File Properties → Details → **Product version** | `0.3.0+a1b2c3d` — version and short commit SHA |
| …built with uncommitted changes | `0.3.0+a1b2c3d.dirty` |
| P21 rule picker (rule description) | Description ends with `[0.3.0+a1b2c3d]` |
| First line of every debug log | `Build: 0.3.0+a1b2c3d` |
| Assembly metadata `CommitSha` | Full 40-character SHA |

- `+a1b2c3d` maps straight to the source:
  `https://github.com/wayne-pipe-and-supply/<repo>/commit/a1b2c3d`.
- **`.dirty` builds are allowed in Play only.** Their source isn't in Git, so they
  can't be reproduced. Before dropping a build into Play that you may want to come
  back to, commit it (`git commit -am "wip: …"` is fine — work-in-progress commits
  are squashed away at merge).
- **Play2 and production run only CI-built release artifacts.** The `release` workflow
  refuses to attach a DLL that isn't stamped with the tagged commit, or that is dirty.
- Audit any rules folder in one line:

  ```powershell
  Get-ChildItem <rules folder>\WPS.P21.*.dll |
      Select-Object Name, @{ n = 'Build'; e = { $_.VersionInfo.ProductVersion } }
  ```

### 5.2 Dev drops while the host holds a DLL lock

*Open item, pending a Play test.* The P21 host locks a rule DLL once loaded, so
iterating in Play means dropping in a new build alongside the locked one instead of
restarting the middleware. Because links bind to the rule **name**, the open question
is how the host behaves when two DLLs in the folder expose the same rule name. The
build stamp answers that directly:

1. In Play, with build A of a rule loaded and linked, drop build B (a different
   commit) under a different file **and** assembly name, with the **same** rule name.
2. Trigger the rule. Check which stamp the debug log and the rule picker show.

| Result | Dev-drop convention |
|---|---|
| B runs, link intact | Name dev drops `WPS.P21.<Rule>.<sha7>.dll`, rule name unchanged. A helper script builds them. |
| A keeps running, or an error/ambiguity | Dev drops need a restart, or a dev-only rule name. Decide which, with the cost of re-linking in mind. |

Release artifacts (Play2 and production) keep the plain `WPS.P21.<Rule>.dll` name either
way; there, a version change is deployed with a service restart in a maintenance
window.

## 6. Environments and promotion

| Environment | Purpose | What may be deployed |
|---|---|---|
| **Play** | Development and experiments | Any build of a working branch |
| **Play2** | Pre-production canary; runs the same P21 build as production | Only a **pre-release** artifact: `vX.Y.Z-rc.N` |
| **Production** | Live | Only a **release** artifact: `vX.Y.Z`, the same DLL that passed Play2 |

Promotion steps:

1. Develop on a working branch; deploy branch builds to **Play** as often as needed.
2. Open the PR. Merge once reviewed and the unit tests pass.
3. Release PR: bump `<Version>`, move `CHANGELOG.md` entries under the new version.
   Merge.
4. Create a GitHub **pre-release** `vX.Y.Z-rc.1` from `main`. The template's `release`
   workflow builds the DLL and attaches it with a SHA-256 checksum. Deploy **that
   file** to **Play2**.
5. Run the canary checks in the PR template. Record the results in the pre-release
   notes. If changes are needed, fix on a branch and cut `-rc.2`.
6. Confirm debug logging will be off in production. Logging must be switched at
   **runtime**, not compiled in, because the production DLL is the same file that
   ran in Play2 (see §6.1).
7. Create the GitHub **release** `vX.Y.Z` on the same commit. The `release` workflow
   attaches **the same DLL** that passed Play2 — it refuses if there is no `-rc` or if
   the commits differ. Deploy it to **production**.
8. Record the deployment: add the environment and date under the version's entry in
   `CHANGELOG.md`, e.g. `Deployed: Play2 2026-10-20, Production 2026-10-27`.

### 6.1 Debug logging

Per-execution debug logs are encouraged in Play and Play2 and forbidden in
production. Because the same DLL moves between them, the switch must be outside the
DLL. The template's `DebugLog` logs only when a marker file named
`<AssemblyName>.debug` sits next to the rule DLL: create it in Play/Play2, never in
production. A rule that uses a compile-time flag (`bool IsDebugModeOn = true`) can't
be promoted unchanged and must move to the runtime switch before its first release.

## 7. Rollback

Every README states its rollback. The baseline:

1. **Immediate:** unassign the rule in Business Rule Organizer. The window behaves as
   if the rule doesn't exist.
2. **Restore:** deploy the previous release's DLL from its GitHub Release and
   reassign. Some P21 hosts lock rule DLLs once loaded, so replacing the file can
   require a P21 service restart — plan for it.
3. Record the rollback in `CHANGELOG.md` and open an issue for the fix.

## 8. Review checklist

Reviewers of a rule PR check, in addition to the core review:

- [ ] Registration in the README matches what the code expects (event, fields, aliases).
- [ ] Every field read is null/blank-safe and trimmed where P21 pads fixed-width values.
- [ ] Failure behavior is deliberate and documented. Rules on print/form paths **fail
      open** (an error must never stop a document printing). Validators state whether
      an unexpected error blocks the save, and show the user a clear message, not a
      raw exception.
- [ ] No business data hard-coded that will change (customer IDs, routes, amounts)
      without a comment saying who owns the list and how it changes, or move it to
      configuration.
- [ ] Debug logging is switched at runtime (§6.1), not by a compile-time flag.
- [ ] User-facing messages say what's wrong and what to do.
- [ ] Unit tests cover the decision logic, including the blocking cases.
- [ ] `CHANGELOG.md` updated under `[Unreleased]`.

## 9. Changes

| Version | Date | Change |
|---|---|---|
| 0.1 | 2026-10-13 | Initial draft. |
