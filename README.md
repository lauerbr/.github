# .github

Shared, reusable GitHub Actions workflows for the `lauerbr` (Ground Orbit
Networks) estate. There is no application code here and nothing in this
repository is deployed — it is a library that other repositories call.

Its settings are managed declaratively by
[`github-as-code`](https://github.com/lauerbr/github-as-code), from
`repos/.github.yaml`. Change repository settings there, not in this
repository's Settings page, or the next `terraform apply` reverts them.

## What is here

| Workflow | Called as |
|---|---|
| `ci-gate` | `lauerbr/.github/.github/workflows/ci-gate.yml@main` |
| `repo-backup` | `lauerbr/.github/.github/workflows/repo-backup.yml@main` |

The doubled `.github/.github/` is correct: the first is this repository's
name, the second is the `.github/` directory that GitHub requires every
workflow file to live in. It reads like a typo. It is not.

`ci-gate` is the one job that always reports a status check, so a repository
whose real CI is path-filtered can still make a check required without
deadlocking itself. Read the header of `.github/workflows/ci-gate.yml` before
editing it — the constraints in there are each the fix for a specific way of
bricking a repository, and now that the definition is shared, breaking one
breaks every caller at once.

`repo-backup` archives `HEAD` as a zip and uploads it to the caller's S3
backup bucket. Callers own the schedule; this file is `workflow_call`-only.
OIDC and the bucket live in the **caller** (the first caller is
`ground-orbit-website` bootstrap). This repository does not assume AWS
roles. Adding a new caller means that repository gets a three-line caller
workflow plus its `main` subject listed on the backup role trust policy
(`StringEquals`, no wildcards). Do not add an `environment:` key on the
reusable job — that would change the OIDC `sub` to `environment:...`, and
the backup role is pinned to `ref:refs/heads/main`.

## The setting that makes this repository work, and is managed by nothing

A **private** repository's reusable workflows are invisible to every other
repository until its Actions access is widened past the default of `none`:

```bash
gh api repos/lauerbr/.github/actions/permissions/access
# {"access_level":"user"}
```

`user` means "accessible from repositories owned by the user", which is the
whole estate. It was set on 2026-08-18.

No Terraform resource in `github-as-code` covers this endpoint —
`github_actions_repository_permissions` and
`github_workflow_repository_permissions` manage Settings → Actions → General,
which is a different surface. So this is a live setting that no manifest
describes and nothing will restore. If a caller ever fails with a "workflow
was not found" error and the path is provably correct, check this first: the
failure is reported against the caller and says nothing about this
repository.

## Consumers

None yet. `aws-network`, `aws-organizations`, `ground-orbit-website` and
`coding-agents-agentcore` each still carry their own local copy of
`ci-gate.yml`. Migrating them is deliberately separate work, because a called
workflow reports as `<caller job name> / <called job name>` rather than as
`ci-gate`, and those four manifests require the literal string `ci-gate` — so
switching a repository over without changing its manifest in the right order
locks it. The sequence is in the workflow header.
