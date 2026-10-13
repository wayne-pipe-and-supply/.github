# .github

Organization-wide defaults and engineering standards for `wayne-pipe-and-supply`.

> **This repository is public.** GitHub only applies default files from an
> organization's `.github` repository when it is public. Never add internal host or
> server names, URLs, credentials, customer information, or vendor-confidential
> details here.

## Standards

| Document | What it covers |
|---|---|
| [Version Control Standards](policies/version-control.md) | Repositories, branches, commits, PRs, releases, and what never goes in Git. Applies to every repository. |
| [P21 Business Rules addendum](policies/p21-business-rules.md) | Extra rules for Prophet 21 business rule repositories. |

## Defaults this repository provides

GitHub uses these in any organization repository that doesn't have its own file of
the same type:

| File | Purpose |
|---|---|
| [`CONTRIBUTING.md`](CONTRIBUTING.md) | Day-to-day branch → PR → merge workflow |
| [`SECURITY.md`](SECURITY.md) | What to do if a secret or customer data is committed |
| [`.github/PULL_REQUEST_TEMPLATE.md`](.github/PULL_REQUEST_TEMPLATE.md) | Default PR description |
| [`.github/ISSUE_TEMPLATE/`](.github/ISSUE_TEMPLATE/) | Bug report and change request forms |
| [`workflow-templates/`](workflow-templates/) | Starter workflows shown under **Actions → New workflow** (PR title check) |

`CODEOWNERS`, `.gitignore`, and `.gitattributes` can't be inherited. Each repository
needs its own; template repositories such as `wps-p21-rule-template` include them.

## Changing anything here

Open a PR with a Conventional Commit title (for example
`docs(policy): clarify release tagging`). The IT Manager approves changes.
