# Addendum: Prophet 21 Business Rules

| | |
|---|---|
| **Extends** | [Version Control Standards](version-control.md) |
| **Applies to** | Repositories containing Prophet 21 (P21) business rule DLLs |
| **Owner** | IT Manager |
| **Status** | Draft v0.2 |

Everything in the core policy applies. This addendum adds what is specific to P21
business rules: one compiled DLL per rule, linked to a field in the P21 window where it
runs (right-click → DynaChange™ Rules → Create Business Rule), triggered by a P21
window event, tested in a Play environment, and then released to production. Business
Rule Organizer reviews, deactivates, and exports links; it does not create them.

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
- **Test the adapter too.** `Rule.Initialize(xml, null)` builds a real
  `DataCollection` from the same `fieldList` XML P21 sends, so the adapter can be run
  in a unit test: aliases, which field fired, field writes, and the unexpected-error
  path. The template has an example.
- **Deploy Release builds only.** Debug builds never leave a developer's machine.

### 2.1 P21.Extensions behaviors to code for

Checked against the SDK source and in testing. The template's adapter follows them.

| Behavior | So |
|---|---|
| `Data.Fields.GetFieldByAlias` throws `KeyNotFoundException` for an alias the link doesn't pass; it never returns null | Don't null-check; let the unexpected-error path handle a mis-linked rule |
| `Data.TriggerColumn` may hold the alias instead of the column name | A rule linked on several fields finds the one that fired from each field's own `DataField.TriggerColumn` (`Y`/`N`) |
| `Data.SetFieldCascade(name, false)` looks the field up by **FieldName**, not alias | Pass `field.FieldName`. The alias throws for fields whose alias differs |
| `RuleResult.Message` is shown to the user even when `Success` is true (seen for Converters) | A rule that repairs a value can say why. `ShowResponse` controls `ResponseAttributes`, not the message |
| P21 pads fixed-width `char` values and trims typed input that is only spaces (no Field Edit fires) | Trim before comparing; don't expect a whitespace-only value from the screen |
| A field's saved blank may be NULL or `''`, depending on how it became blank | SQL that reports on it uses `NULLIF(LTRIM(RTRIM(col)), '') IS NULL` |

### 2.2 Choosing the Rule Type

Verified in a test environment, 2026-10-13/14, on the Order Entry Item Creation popup.

| Rule Type | Runs on Field Edit | When it returns `Success = false` |
|---|---|---|
| **Converter** | Before P21's own validation of the field | The edit is rejected. Seen with an empty previous value: the field stayed empty, and the lookup description beside it cleared |
| **Validator** | After P21's validation (or on Save) | The edit is rejected and the field goes back to its previous value; the lookup description beside it keeps the rejected value's text |

- **P21 runs only one Validator per field.** A second Validator on the same field is
  saved and passes Run Test, but never runs live. Before linking a Validator, check the
  field with **View All Business Rules for Window**; if it already has one, link the
  new rule as a Converter. The Converter runs first; when it rejects, the field's
  Validator doesn't run for that edit.
- **Save does not fire on a popup's OK.** A Validator on Save let bad data through the
  Item Creation popup. Link rules on popups with Field Edit.
- **Field Edit fires however the user leaves the field:** Tab, choosing from a lookup,
  Enter, or clicking a button straight after typing. A checkbox fires on the click.
- **Run Test can't prove a link runs.** It calls the rule whatever the Rule Type and
  with whatever is in the test pane. Only a live test, checked against the debug log,
  shows that P21 calls the link (§6.2).
- **Changes in Business Rule Organizer** (Apply Changes) take effect in open windows
  immediately.
- A rule may be linked on several fields of a window (one row per link). The README
  lists every link, and rollback deactivates all of them or none (§7).

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
rule's first release candidate (§6):

| Section | Contents |
|---|---|
| Purpose | The business problem, in words a non-developer understands |
| Status | Which environments it is in, and the version in each |
| Registration | Where it is linked (window, tab and data window, linked field, or one row per link if there are several), the other rules on the same fields, the Configuration Options values as P21 names them (Rule Type, Apply Rule On, Run Type, Multi-Row, Global Rule, Trigger Rule in OE Windows), a step-by-step "How to link it", and the live-test results |
| Required fields | Every Field Selector row the rule needs: data window, field, and the exact **Pass To Rule As** alias |
| Behavior | What it does when the rule passes, when it blocks, and when something unexpected happens (fails open or fails closed, and why) |
| Configuration | Anything that may change without a code change (and where it lives), or an explicit "none" |
| Build and test | Commands from a clean clone |
| Deploy | Step by step, including where the DLL goes and linking it as in Registration |
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
- **`.dirty` builds are allowed in the test environments (Play, Play2) only.** Their
  source isn't in Git, so they can't be reproduced. Before dropping a build you may
  want to come back to, commit it (`git commit -am "wip: …"` is fine —
  work-in-progress commits are squashed away at merge).
- **Production runs only CI-built release artifacts.** The `release` workflow refuses
  to attach a DLL that isn't stamped with the tagged commit, or that is dirty.
- Audit any rules folder in one line:

  ```powershell
  Get-ChildItem <rules folder>\WPS.P21.*.dll |
      Select-Object Name, @{ n = 'Build'; e = { $_.VersionInfo.ProductVersion } }
  ```

### 5.2 Dev drops while the host holds a DLL lock

*Open item, pending a test.* The P21 host locks a rule DLL once loaded, so iterating
in a test environment means dropping in a new build alongside the locked one instead
of restarting the middleware. Because links bind to the rule **name**, the open question
is how the host behaves when two DLLs in the folder expose the same rule name. The
build stamp answers that directly:

1. In Play or Play2, with build A of a rule loaded and linked, drop build B (a different
   commit) under a different file **and** assembly name, with the **same** rule name.
2. Trigger the rule. Check which stamp the debug log and the rule picker show.

Observations so far:

- 2026-10-13, before any link existed: with an old and a new DLL both exposing the same
  rule name, the Create Business Rule dialog's **Rules** tab listed two rows with the
  same Rule Name and different Assembly Names. Which one a saved link would run is still
  untested; until it is, remove the older DLL before linking.
- 2026-10-14: a DLL with a file name the host had never loaded appeared in the Rules
  tab without a restart. Only replacing or deleting a loaded file needs one.

| Result | Dev-drop convention |
|---|---|
| B runs, link intact | Name dev drops `WPS.P21.<Rule>.<sha7>.dll`, rule name unchanged. A helper script builds them. |
| A keeps running, or an error/ambiguity | Dev drops need a restart, or a dev-only rule name. Decide which, with the cost of re-linking in mind. |

Release artifacts (release candidates and releases) keep the plain
`WPS.P21.<Rule>.dll` name either way; in production, a version change is deployed with
a service restart in a maintenance window.

## 6. Environments and promotion

**Play** and **Play2** are parallel test environments. Neither is a stage after the
other: a rule is developed and tested in whichever one the work uses, then released
to production. Nothing is promoted from Play to Play2.

| Environment | Purpose | What may be deployed |
|---|---|---|
| **Play**, **Play2** | Development and testing | Any build: working-branch builds while developing; the **pre-release** artifact `vX.Y.Z-rc.N` for the final check before production |
| **Production** | Live | Only a **release** artifact: `vX.Y.Z`, the same DLL as the release candidate that passed testing |

Release steps:

1. Develop on a working branch; deploy branch builds to Play or Play2 as often as
   needed.
2. Open the PR. Merge once reviewed and the unit tests pass.
3. Release PR: bump `<Version>`, move `CHANGELOG.md` entries under the new version.
   Merge.
4. Create a GitHub **pre-release** `vX.Y.Z-rc.1` from `main`. The template's `release`
   workflow builds the DLL and attaches it with a SHA-256 checksum. Deploy **that
   file** to the test environment (Play or Play2).
5. Test it as in §6.2. Record the environment and results in the pre-release notes
   and in the README. If code changes are needed, fix on a branch and cut `-rc.2`. A
   change to a link only (for example its Rule Type) needs no new rc; record it in the
   README and the notes.
6. Confirm debug logging will be off in production. Logging must be switched at
   **runtime**, not compiled in, because the production DLL is the same file that
   was tested (see §6.1).
7. Create the GitHub **release** `vX.Y.Z` on the same commit. The `release` workflow
   attaches **the same DLL** as the tested release candidate — it refuses if there is
   no `-rc` or if the commits differ. Docs-only PRs merged after the rc (test results,
   README fixes) don't change this: the release still goes on the rc's commit, not on
   the newest `main`. Deploy it to **production**.
8. Record the deployment: add the test environment and production dates under the
   version's entry in `CHANGELOG.md`, e.g.
   `Tested: Play 2026-10-20 (rc.1). Deployed: Production 2026-10-27`.

### 6.1 Debug logging

Per-execution debug logs are encouraged in Play and Play2 and forbidden in
production. Because the same DLL goes from testing to production, the switch must be
outside the DLL. The template's `DebugLog` logs only when a marker file named
`<AssemblyName>.debug` sits next to the rule DLL: create it in Play/Play2, never in
production. A rule that uses a compile-time flag (`bool IsDebugModeOn = true`) can't
be promoted unchanged and must move to the runtime switch before its first release.

The rule checks for the marker on every run, so adding or deleting it needs no restart.
Delete it when testing ends; keep the logs as long as they are useful.

Each run's log must record, from its first lines: the build stamp, the user, the
link's Rule Type and Apply Rule On, which field fired, every field read (value,
original value, TriggerColumn), every field written, and the decision. Then a missing
log file means P21 never called the link, which no screen shows.

### 6.2 Testing a release candidate

1. **Debug marker on**, and the Play/Play2 logs folder readable by whoever tests.
2. **Go/no-go cases first:** the ones the design can't work without. For a Field Edit
   link that includes every way of leaving the field (Tab, lookup, Enter, clicking a
   button straight after typing). If one fails, stop and take it back to the design
   before testing anything else.
3. Then every row of the README's Behavior table, through the real window. Start each
   case from a fresh window or popup.
4. **After each case, read the log**, and confirm the rule ran, on the expected link,
   with the expected decision. A result that looks right on screen but has no log line
   is a failure.
5. Where the rule's outcome is a saved value, check what was saved, with a read-only
   query the IT Manager approves.
6. Record each case (pass, or what happened) in the pre-release notes and the README.

## 7. Rollback

Every README states its rollback. The baseline:

1. **Immediate:** in Business Rule Organizer, set the rule's **Row Status** to
   **Inactive** and click **Apply Changes**. The window behaves as if the rule doesn't
   exist, and the link (fields, aliases, options) is kept for the restore. A rule
   linked on several fields has one row per link: deactivate all of them. Deactivating
   only some leaves half a rule.
2. **Restore:** deploy the previous release's DLL from its GitHub Release and set the
   row back to **Active**. P21 hosts lock rule DLLs once loaded, so replacing the file
   means stopping the P21 site or service — plan for it.
3. Record the rollback in `CHANGELOG.md` and open an issue for the fix.

## 8. Review checklist

Reviewers of a rule PR check, in addition to the core review:

- [ ] Registration in the README matches what the code expects (linked field, Field
      Selector rows, Pass To Rule As aliases, Configuration Options).
- [ ] No Validator on a field that already has one (§2.2); the README names the other
      rules on the same fields.
- [ ] A rule linked on several fields finds the field that fired from each field's
      `TriggerColumn` (§2.1), and the README lists every link.
- [ ] The adapter has tests (§2), and the debug log records what §6.1 lists.
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
| 0.2 | 2026-10-14 | Lessons from the first two rules tested end to end: P21.Extensions behaviors (§2.1), choosing the Rule Type and one Validator per field (§2.2), adapter tests, what the debug log records (§6.1), how to test a release candidate (§6.2), new DLL file names load without a restart (§5.2), the release stays on the rc's commit after docs PRs, multi-link rollback. |
