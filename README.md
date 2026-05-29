# Renovate orphan-branch deletion bug with empty branchPrefix

## Current behavior

When `branchPrefix` is set to `""` (empty string) together with a custom `branchName` template,
Renovate treats **every branch in the repository** as a branch it manages.
As a result it deletes branches it cannot match to an active update as "orphan" branches —
including the configured `baseBranch` itself and unrelated feature/bugfix branches.

### Reproduction steps

The repository has the following branch structure (enforced by GitHub branch-protection rules):

```
^(main|\d{3}[\dxX]*)/(develop|renovate-[-\w]+|release|(?:(presc|admin|ddm|pharma|devops|archi)/)?(?:(?:multi-)?feature|bugfix|quality|experiment|buildfix|i18n|patch)(?:/(.+)))|PR-\d+|gh-readonly-queue/.+$
```

Example branches:
| Branch | Purpose |
|---|---|
| `main/develop` | Default / master branch |
| `main/feature/EXAMPLE-123` | Feature branch (not managed by Renovate) |
| `main/quality/renovate-maven-artifacts` | Renovate-managed update branch |

Because `branchPrefix: ""` matches every string, Renovate considers all branches above as
candidates it manages. When it finds `main/develop` or `main/feature/EXAMPLE-123` with no
corresponding open update, it deletes them as "orphans".

Observed in logs (repository name obfuscated):

```
INFO: Deleting orphan branch  branch=main/develop
INFO: Deleting orphan branch  branch=main/multi-feature/HEPIC-12345-split
```

### Configuration (`renovate.json` on `main/develop`)

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "baseBranches": ["main/develop"],
  "branchPrefix": "",
  "additionalBranchPrefix": "",
  "branchName": "{{lookup (split baseBranch '/') 0 }}/quality/renovate-{{{branchTopic}}}"
}
```

The empty `branchPrefix` is required because the desired Renovate branch names start with
the version prefix (e.g. `main/`, `400XXXX/`), not with a fixed `renovate/` prefix.

## Expected behavior

Renovate should only consider a branch as "orphan" if it was **created by Renovate** (i.e. its
name was generated from the configured `branchName` template). Branches that do not match the
generated pattern — such as `main/develop` (the `baseBranch`) or `main/feature/EXAMPLE-123`
(an unrelated feature branch) — must **never** be deleted, regardless of whether `branchPrefix`
is empty.

## Link to the Renovate issue or Discussion

> Will be added once the discussion is created.
