# synlearn-checks

The reusable GitHub Actions workflow that grades SynLearn DevOps Associate labs.
Learner repositories (made from the `devops-associate-labs` template) call it from
`.github/workflows/synlearn-graded.yml`:

```yaml
jobs:
  grade:
    permissions:
      id-token: write
      contents: read
    uses: Syntera-labs-sa/synlearn-checks/.github/workflows/check.yml@v1.0.0
    with:
      labs-file: .synlearn/graded-labs
```

## Why the checks live here and not in the learner's repository

A learner owns their repository and can change any file in it, including workflows.
If the grading steps lived there, a learner could replace them with `echo PASS`.

Here, learners have no write access. When their workflow calls this one, GitHub puts the
path and ref of **this** file in the `job_workflow_ref` claim of the OIDC token, for example
`Syntera-labs-sa/synlearn-checks/.github/workflows/check.yml@refs/tags/v1.0.0`. GitHub signs that
token, so the SynLearn server can prove the results came from this workflow at a release it
trusts, and credit them to the owner of the repository (`repository_owner` claim).

The check scripts themselves are served by SynLearn (`GET /api/labs/<id>/graded-check`),
so fixing a check does not need a new release here. This workflow only fetches, runs,
collects and reports.

## How it works

| Job | Permissions | What it does |
|---|---|---|
| `check` | `contents: read` | Checks out the learner's repo, fetches each lab's graded check, runs it with a clean environment and a 180 s timeout, parses `PASS`/`FAIL` lines. |
| `submit` | `id-token: write` | Never checks out learner code. Requests an OIDC token with audience `synlearn` and POSTs the results to `/api/labs/graded`. Fails the run if any lab has not passed, so the learner sees a red cross. |

The API URL is fixed in the workflow (not an input), so a caller cannot point grading at another server.

## Rules for graded check scripts

- **Inspect, never execute, learner code.** Read files, parse YAML/HCL/JSON, or run
  `terraform fmt -check`. Do not run `docker build`, `terraform init`/`plan`,
  `ansible-playbook` or any script from the learner's repo (those belong in the practice
  checks run by `synlearn check` in the codespace). Anything a check executes can tamper with the job's outputs. The worst it can do is
  forge a pass for the learner's own account (the `submit` job and its token are out of
  reach), but graded checks should still be static.
- Print one `PASS <message>` or `FAIL <message>` line per check; pass only if every line is
  `PASS` and the script exits 0 (the SynLearn terminal-lab contract).
- Run from the repository root; `$SYNLEARN_REPO_ROOT` holds its path.

## Releases must be pinned

- Tag every release (`v1.0.0`, `v1.1.0`, ...) and protect tags with a ruleset so tags cannot be moved or deleted.
- The template's workflow pins a tag (or a full commit SHA). Never `@main`.
- The SynLearn server keeps an allowlist of accepted refs (for example
  `refs/tags/v1.0.0`). Add the new tag to the allowlist when you release, and remove old
  tags only after learners have had time to update.
- Third-party actions in `check.yml` are pinned to full commit SHAs.
