---
on:
  schedule:
    - cron: '0 9 1 * *'  # 09:00 UTC on the 1st of each month
  workflow_dispatch:

permissions:
  copilot-requests: write
  contents: read
  issues: read
  pull-requests: read

engine: copilot

tools:
  github:
    toolsets: [default]

network: defaults

steps:
  - name: Checkout repo with full history
    uses: actions/checkout@v7
    with:
      fetch-depth: 0
      persist-credentials: false

  - name: Preflight upstream graph and isolated rebase
    id: repo_preflight
    shell: bash
    run: |
      set -euo pipefail
      mkdir -p "${GITHUB_WORKSPACE}/.gh-aw"

      git remote get-url upstream >/dev/null 2>&1 || git remote add upstream https://github.com/mmistakes/minimal-mistakes.git
      git fetch --prune --no-tags origin '+refs/heads/*:refs/remotes/origin/*'
      git fetch --prune --no-tags upstream '+refs/heads/*:refs/remotes/upstream/*'

      fork_head_sha="$(git rev-parse origin/bex-master)"
      old_base_sha="$(git merge-base origin/bex-master upstream/master)"
      upstream_head_sha="$(git rev-parse upstream/master)"
      upstream_commit_count="$(git rev-list --count "${old_base_sha}..upstream/master")"

      rebase_status="clean"
      rebase_log="${GITHUB_WORKSPACE}/.gh-aw/upstream-softfork-rebase.txt"
      conflict_files="${GITHUB_WORKSPACE}/.gh-aw/upstream-softfork-conflicts.txt"
      conflict_diff="${GITHUB_WORKSPACE}/.gh-aw/upstream-softfork-conflict-diff.txt"
      : > "${rebase_log}"
      : > "${conflict_files}"
      : > "${conflict_diff}"
      if [ "${upstream_commit_count}" -gt 0 ]; then
        worktree_dir="$(mktemp -d "${RUNNER_TEMP}/upstream-softfork.XXXXXX")"
        if ! git worktree add --detach "${worktree_dir}" origin/bex-master >"${rebase_log}" 2>&1; then
          rebase_status="conflict"
        else
          git -C "${worktree_dir}" config user.name "github-actions[bot]"
          git -C "${worktree_dir}" config user.email "41898282+github-actions[bot]@users.noreply.github.com"
          if ! git -C "${worktree_dir}" rebase --onto upstream/master "${old_base_sha}" >>"${rebase_log}" 2>&1; then
            rebase_status="conflict"
            git -C "${worktree_dir}" diff --name-only --diff-filter=U > "${conflict_files}"
            git -C "${worktree_dir}" diff > "${conflict_diff}"
          fi
          git worktree remove --force "${worktree_dir}" >/dev/null 2>&1 || true
        fi
      fi

      git log --reverse --format='%h %s' "${old_base_sha}..origin/bex-master" > "${GITHUB_WORKSPACE}/.gh-aw/upstream-softfork-private-log.txt"
      git cherry upstream/master origin/bex-master > "${GITHUB_WORKSPACE}/.gh-aw/upstream-softfork-cherry.txt"
      git diff "${old_base_sha}..origin/bex-master" > "${GITHUB_WORKSPACE}/.gh-aw/upstream-softfork-private-diff.txt"
      git diff "${old_base_sha}..upstream/master" > "${GITHUB_WORKSPACE}/.gh-aw/upstream-softfork-upstream-diff.txt"

      jq -n \
        --arg fork_head_sha "${fork_head_sha}" \
        --arg old_base_sha "${old_base_sha}" \
        --arg upstream_head_sha "${upstream_head_sha}" \
        --arg upstream_commit_count "${upstream_commit_count}" \
        --arg rebase_status "${rebase_status}" \
        '{fork_head_sha:$fork_head_sha, old_base_sha:$old_base_sha, upstream_head_sha:$upstream_head_sha, upstream_commit_count:($upstream_commit_count|tonumber), rebase_status:$rebase_status}' \
        > "${GITHUB_WORKSPACE}/.gh-aw/upstream-softfork-preflight.json"

      echo "fork_head_sha=${fork_head_sha}" >> "${GITHUB_OUTPUT}"
      echo "old_base_sha=${old_base_sha}" >> "${GITHUB_OUTPUT}"
      echo "upstream_head_sha=${upstream_head_sha}" >> "${GITHUB_OUTPUT}"
      echo "upstream_commit_count=${upstream_commit_count}" >> "${GITHUB_OUTPUT}"
      echo "rebase_status=${rebase_status}" >> "${GITHUB_OUTPUT}"

post-steps:
  - name: Notify upstream rebase webhook
    if: always()
    env:
      FORK_HEAD_SHA: ${{ steps.repo_preflight.outputs.fork_head_sha }}
      OLD_BASE_SHA: ${{ steps.repo_preflight.outputs.old_base_sha }}
      UPSTREAM_HEAD_SHA: ${{ steps.repo_preflight.outputs.upstream_head_sha }}
      UPSTREAM_COMMIT_COUNT: ${{ steps.repo_preflight.outputs.upstream_commit_count }}
      REBASE_STATUS: ${{ steps.repo_preflight.outputs.rebase_status }}
      REBASE_WEBHOOK_URL: ${{ secrets.REBASE_WEBHOOK_URL }}
    run: |
      set -euo pipefail
      result_file="${GITHUB_WORKSPACE}/.gh-aw/upstream-softfork-monitor-result.json"

      if [ -z "${REBASE_WEBHOOK_URL:-}" ]; then
        echo "REBASE_WEBHOOK_URL secret is not configured — skipping webhook notification"
        exit 0
      fi
      if [ ! -f "${result_file}" ]; then
        echo "No machine-readable result file found — skipping webhook"
        exit 0
      fi
      if [ -z "${FORK_HEAD_SHA:-}" ] || [ -z "${OLD_BASE_SHA:-}" ] || [ -z "${UPSTREAM_HEAD_SHA:-}" ] || [ -z "${UPSTREAM_COMMIT_COUNT:-}" ]; then
        echo "Preflight SHA outputs are missing — skipping webhook"
        exit 0
      fi
      if [ "${UPSTREAM_COMMIT_COUNT}" -le 0 ]; then
        echo "No upstream commits — skipping webhook"
        exit 0
      fi
      if [ "${REBASE_STATUS}" != "clean" ]; then
        echo "Rebase is not clean — skipping webhook"
        exit 0
      fi

      if ! jq -e '
        type == "object" and
        .verdict == "retained" and
        (.uncertain | type) == "boolean" and
        .uncertain == false
      ' "${result_file}" >/dev/null; then
        echo "Semantic verdict is not clean retained — skipping webhook"
        exit 0
      fi

      git fetch --prune --no-tags origin '+refs/heads/*:refs/remotes/origin/*'
      git fetch --prune --no-tags upstream '+refs/heads/*:refs/remotes/upstream/*'
      actual_fork_head="$(git rev-parse origin/bex-master)"
      actual_old_base="$(git merge-base origin/bex-master upstream/master)"
      actual_upstream_head="$(git rev-parse upstream/master)"
      actual_upstream_commit_count="$(git rev-list --count "${actual_old_base}..upstream/master")"
      if [ "${actual_fork_head}" != "${FORK_HEAD_SHA}" ] || [ "${actual_old_base}" != "${OLD_BASE_SHA}" ] || [ "${actual_upstream_head}" != "${UPSTREAM_HEAD_SHA}" ] || [ "${actual_upstream_commit_count}" != "${UPSTREAM_COMMIT_COUNT}" ]; then
        echo "Git refs changed since preflight — skipping webhook"
        exit 0
      fi

      message="${GITHUB_REPOSITORY}: upstream changed, rebase clean, retained private patch; run=${GITHUB_SERVER_URL}/${GITHUB_REPOSITORY}/actions/runs/${GITHUB_RUN_ID}"
      payload="$(jq -n \
        --arg message "${message}" \
        --arg repository "${GITHUB_REPOSITORY}" \
        --arg run_url "${GITHUB_SERVER_URL}/${GITHUB_REPOSITORY}/actions/runs/${GITHUB_RUN_ID}" \
        --arg old_base_sha "${OLD_BASE_SHA}" \
        --arg upstream_head_sha "${UPSTREAM_HEAD_SHA}" \
        --arg upstream_commit_count "${UPSTREAM_COMMIT_COUNT}" \
        '{message:$message, repository:$repository, run_url:$run_url, old_base_sha:$old_base_sha, upstream_head_sha:$upstream_head_sha, upstream_commit_count:($upstream_commit_count|tonumber)}')"
      curl --fail --silent --show-error --max-time 10 -H "Content-Type: application/json" -d "${payload}" "${REBASE_WEBHOOK_URL}"

safe-outputs:
  create-issue:
    max: 5
  update-issue:
    max: 5
---

# upstream-softfork-monitor

Track upstream changes to `mmistakes/minimal-mistakes` while preserving the private `bex-master` stack. This workflow answers one narrow question: did upstream replace private patch work, and if so, what should be recorded?

## Deterministic pre-agent steps

1. Check out the repo with full history, add `upstream` if needed, and fetch both remotes.
2. Read the fork from the fetched `origin/bex-master` remote-tracking ref.
3. Compute immutable pre-agent values:
   - `fork_head_sha = git rev-parse origin/bex-master`
   - `old_base_sha = git merge-base origin/bex-master upstream/master`
   - `upstream_head_sha = git rev-parse upstream/master`
   - `upstream_commit_count = git rev-list --count "${old_base_sha}..upstream/master"`
   - `rebase_status = clean|conflict` from a disposable worktree rebase attempt
4. Save deterministic evidence before the agent:
   - `${GITHUB_WORKSPACE}/.gh-aw/upstream-softfork-preflight.json`
   - `${GITHUB_WORKSPACE}/.gh-aw/upstream-softfork-private-log.txt`
   - `${GITHUB_WORKSPACE}/.gh-aw/upstream-softfork-cherry.txt`
   - `${GITHUB_WORKSPACE}/.gh-aw/upstream-softfork-private-diff.txt`
   - `${GITHUB_WORKSPACE}/.gh-aw/upstream-softfork-upstream-diff.txt`
   - `${GITHUB_WORKSPACE}/.gh-aw/upstream-softfork-rebase.txt`
   - `${GITHUB_WORKSPACE}/.gh-aw/upstream-softfork-conflicts.txt`
   - `${GITHUB_WORKSPACE}/.gh-aw/upstream-softfork-conflict-diff.txt`
5. Expose them as GitHub step outputs and never let the agent rewrite the refs that generated them.

## Agent task

- Read only the precomputed evidence and the current `FORK.md` references.
- Decide whether the private patch is still retained, replaced by upstream, or uncertain.
- Exact equivalence evidence is generated before the agent when possible: use `git cherry`, file diffs, `git log`, and the rebase result captured above.
- Do not write branch refs or mutate the workspace beyond the result file.
- Emit a machine-readable result file at `${GITHUB_WORKSPACE}/.gh-aw/upstream-softfork-monitor-result.json` containing only semantic verdict data needed by the post-step:
  - `verdict`: `retained`, `replaced`, or `uncertain`
  - `uncertain`: true/false
  - `resolution`: short proposed issue resolution
  - `evidence`: short evidence object or list
  - `old_base_sha`
  - `upstream_head_sha`
  - `upstream_commit_count`
- Leave out authoritative graph/rebase booleans from the result file; the post-step must gate on immutable step outputs instead.

## Outcome rules

- No upstream commits: no webhook and no issue.
- Rebase conflict, exact replacement, semantic replacement, or uncertainty: issue path only, no webhook.
- clean + count > 0 + verdict == retained + uncertain == false: webhook allowed.
- Never expose `REBASE_WEBHOOK_URL` in the agent context or logs.
- Never create or update an issue for the clean+retained case.
- Use `safe-outputs` issue creation/update only for changed-upstream outcomes that are not clean+retained.

## Final summary

Keep the final job summary small: base SHA, upstream head SHA, upstream commit count, rebase status, verdict, issue URL if created, and webhook result.
