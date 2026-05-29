# Renovate orphan-branch deletion bug with empty branchPrefix

## Background

A similar problem was previously reported in
[Discussion #15732 — Renovate accidentally deleted another user's branch](https://github.com/renovatebot/renovate/discussions/15732)
and was addressed by
[Issue #15733 — Check if branch modified before pruning](https://github.com/renovatebot/renovate/issues/15733),
released in version **32.83.3**.

The fix introduced an `isModified` check that skips orphan-branch deletion when a branch
contains commits from non-Renovate authors. This works in the common case, but the fix is
**incomplete** when `branchPrefix` is set to `""` (empty string):

- The `baseBranch` itself is **still** targeted for deletion (the `isModified` check is
  bypassed when branch cache is absent, and the deletion is attempted unconditionally).
- Branches that are "clean" relative to base (no extra commits, or commits that Renovate
  cannot attribute to a non-Renovate author) are **still** at risk.
- The root cause — Renovate using `branchPrefix: ""` to match **every branch** in the
  repository as a potentially Renovate-managed branch — remains unaddressed.

This reproduction was confirmed against **renovatebot/github-action v43.0.19**
(Renovate engine ≥ 41.x), well past the 32.83.3 release of the supposed fix.

## Current behavior

When `branchPrefix` is set to `""` (empty string) together with a custom `branchName` template,
Renovate treats **every branch in the repository** as a branch it manages.
As a result it marks branches it cannot match to an active update as "orphan" branches and
attempts to delete them — including the configured `baseBranch` itself and unrelated
feature/bugfix branches.

The only partial safety nets are:
1. GitHub's built-in protection of the **default branch** (refuses `git push --delete`).
2. Renovate's own `isModified` check, which skips deletion if a branch contains commits from
   non-Renovate authors — **but only when branch state can be determined via git**.
   When no cache is present (e.g. first run, or `baseBranch`), the check is bypassed and
   deletion is attempted unconditionally.

### Branch naming convention

The repository enforces the following branch structure via GitHub rules:

```
^(main|\d{3}[\dxX]*)/(develop|renovate-[-\w]+|release|(?:(presc|admin|ddm|pharma|devops|archi)/)?(?:(?:multi-)?feature|bugfix|quality|experiment|buildfix|i18n|patch)(?:/(.+)))|PR-\d+|gh-readonly-queue/.+$
```

Example branches used in this reproduction:

| Branch | Purpose |
|---|---|
| `main/develop` | Default / master branch — also the configured `baseBranch` |
| `main/feature/EXAMPLE-123` | Feature branch — not managed by Renovate |
| `main/multi-feature/HEPIC-12345` | Multi-feature branch — not managed by Renovate |
| `main/quality/renovate-*` | Renovate-managed update branch (correctly created) |

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

The empty `branchPrefix` is required because the desired Renovate branch names start with the
version prefix (e.g. `main/`, `400XXXX/`), not with a fixed `renovate/` prefix.

### Confirmed reproduction results

**Run 1 — first execution (no prior Renovate state)**

Renovate correctly created the update branch:
```
INFO: Branch updated  branch=main/quality/renovate-org.apache.commons-commons-lang3-3.x
```

Renovate then attempted to delete the `baseBranch` as an orphan — no `isModified` check was
performed (cache not present → deletion attempted unconditionally):
```
DEBUG: setCachedModifiedResult(): Branch cache not present  branch=main/develop
INFO:  Deleting orphan branch                               branch=main/develop
DEBUG: Git function thrown
       "commands": ["push", "--delete", "origin", "main/develop", "--no-verify"],
       "message": "refusing to delete the current branch: refs/heads/main/develop"
DEBUG: No remote branch to delete with name: main/develop
```
→ Deletion **blocked only** because `main/develop` is the GitHub default branch.

**Run 2 — second execution (warm state)**

The feature branches were also identified as orphans. Here the `isModified` git check ran and
found commits from non-Renovate authors, so deletion was skipped:
```
DEBUG: branch.isModified(): using git to calculate       branch=main/feature/EXAMPLE-123
DEBUG: branch.isModified() = true
DEBUG: Orphan Branch is modified - skipping branch deletion  branch=main/feature/EXAMPLE-123

DEBUG: branch.isModified(): using git to calculate       branch=main/multi-feature/HEPIC-12345
DEBUG: branch.isModified() = true
       "unrecognizedAuthors": ["<committer-email>"]
DEBUG: Orphan Branch is modified - skipping branch deletion  branch=main/multi-feature/HEPIC-12345
```
→ Deletion **skipped** only because the branches contained commits from non-Renovate authors.

### Why this is insufficient

The `isModified` check is **not a reliable safeguard**:

- It does **not** protect the `baseBranch` (no check is performed — deletion is attempted
  unconditionally when cache is absent).
- It does **not** protect branches that are "clean" relative to the base (e.g. freshly
  created branches, or branches where commits happen to look like Renovate's).
- It is a coincidental side-effect, not an intentional design choice for protecting
  non-Renovate branches.

In the original incident (production repository), both `main/develop` and
`main/multi-feature/HEPIC-12345-split` were successfully deleted:
```
INFO: Deleting orphan branch  branch=main/develop
INFO: Deleting orphan branch  branch=main/multi-feature/HEPIC-12345-split
```

## Expected behavior

Renovate should identify orphan branches **solely** by matching against the configured
`branchName` template (or a combination of `branchPrefix` + `additionalBranchPrefix` +
`branchTopic`). A branch that does not match the generated pattern must **never** be
treated as a Renovate-managed branch, regardless of whether `branchPrefix` is empty.

In particular:
- The configured `baseBranch` must **never** be a candidate for orphan deletion.
- Branches whose names do not match the `branchName` template must **never** be candidates
  for orphan deletion.

## Link to the Renovate issue or Discussion

> Will be added once the discussion is created.
