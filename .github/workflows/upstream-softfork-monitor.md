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
  - name: Submit automated report
    if: always()
    env:
      FORK_HEAD_SHA: ${{ steps.repo_preflight.outputs.fork_head_sha }}
      OLD_BASE_SHA: ${{ steps.repo_preflight.outputs.old_base_sha }}
      UPSTREAM_HEAD_SHA: ${{ steps.repo_preflight.outputs.upstream_head_sha }}
      UPSTREAM_COMMIT_COUNT: ${{ steps.repo_preflight.outputs.upstream_commit_count }}
      REBASE_STATUS: ${{ steps.repo_preflight.outputs.rebase_status }}
      JOB_STATUS: ${{ job.status }}
      NOTIFICATION_URL: ${{ secrets.NOTIFICATION_URL }}
    run: |
      set -euo pipefail
      result_file="${GITHUB_WORKSPACE}/.gh-aw/upstream-softfork-monitor-result.json"

      : "${NOTIFICATION_URL:?NOTIFICATION_URL must be configured}"
      : "${GITHUB_REPOSITORY:?GITHUB_REPOSITORY must be set}"
      : "${GITHUB_RUN_ID:?GITHUB_RUN_ID must be set}"
      : "${GITHUB_RUN_ATTEMPT:?GITHUB_RUN_ATTEMPT must be set}"
      : "${GITHUB_SERVER_URL:?GITHUB_SERVER_URL must be set}"

      heartbeat=false
      outcome="failure"
      verdict="unknown"
      uncertain="unknown"
      resolution=""
      issue_expected=false
      reason=""
      if [ "${JOB_STATUS}" = "success" ] && [ "${UPSTREAM_COMMIT_COUNT:-}" = "0" ]; then
        heartbeat=true
        outcome="success"
      elif [ -z "${FORK_HEAD_SHA:-}" ] || [ -z "${OLD_BASE_SHA:-}" ] || [ -z "${UPSTREAM_HEAD_SHA:-}" ] || [ -z "${UPSTREAM_COMMIT_COUNT:-}" ] || [ -z "${REBASE_STATUS:-}" ]; then
        reason="preflight outputs are missing"
      elif ! [[ "${UPSTREAM_COMMIT_COUNT}" =~ ^[0-9]+$ ]]; then
        reason="upstream commit count is invalid"
      elif [ ! -f "${result_file}" ]; then
        reason="machine-readable agent result is missing"
      elif ! jq -e '
        type == "object" and
        (.verdict | IN("retained", "replaced", "uncertain")) and
        (.uncertain | type) == "boolean"
      ' "${result_file}" >/dev/null; then
        reason="machine-readable agent result is invalid"
      else
        verdict="$(jq -r '.verdict' "${result_file}")"
        uncertain="$(jq -r '.uncertain' "${result_file}")"
        resolution="$(jq -r '.resolution // ""' "${result_file}")"
        outcome="attention_required"
        issue_expected=true
        if [ "${REBASE_STATUS}" = "clean" ] && [ "${verdict}" = "retained" ] && [ "${uncertain}" = "false" ]; then
          outcome="success"
          issue_expected=false
        fi
      fi

      if [ "${heartbeat}" != "true" ] && [ -z "${reason}" ]; then
        if ! git fetch --prune --no-tags origin '+refs/heads/*:refs/remotes/origin/*' ||
          ! git fetch --prune --no-tags upstream '+refs/heads/*:refs/remotes/upstream/*'; then
          outcome="failure"
          reason="git ref revalidation failed"
        else
          actual_fork_head="$(git rev-parse origin/bex-master)"
          actual_old_base="$(git merge-base origin/bex-master upstream/master)"
          actual_upstream_head="$(git rev-parse upstream/master)"
          actual_upstream_commit_count="$(git rev-list --count "${actual_old_base}..upstream/master")"
          if [ "${actual_fork_head}" != "${FORK_HEAD_SHA}" ] || [ "${actual_old_base}" != "${OLD_BASE_SHA}" ] || [ "${actual_upstream_head}" != "${UPSTREAM_HEAD_SHA}" ] || [ "${actual_upstream_commit_count}" != "${UPSTREAM_COMMIT_COUNT}" ]; then
            outcome="failure"
            reason="git refs changed since preflight"
          fi
        fi
      fi
      if [ "${heartbeat}" != "true" ] && [ "${JOB_STATUS}" != "success" ] && [ -z "${reason}" ]; then
        outcome="failure"
        reason="workflow failed before report"
      fi

      event_id="github-actions:${GITHUB_REPOSITORY}:${GITHUB_RUN_ID}:${GITHUB_RUN_ATTEMPT}:minimal-mistakes.upstream-softfork.completed"
      occurred_at="$(date -u +%Y-%m-%dT%H:%M:%SZ)"
      run_url="${GITHUB_SERVER_URL}/${GITHUB_REPOSITORY}/actions/runs/${GITHUB_RUN_ID}"
      payload="$(jq -n \
        --arg workflow "upstream-softfork-monitor" \
        --arg run_url "${run_url}" \
        --arg outcome "${outcome}" \
        --arg reason "${reason}" \
        --arg verdict "${verdict}" \
        --arg uncertain "${uncertain}" \
        --arg rebase_status "${REBASE_STATUS:-unknown}" \
        --arg fork_head_sha "${FORK_HEAD_SHA:-unknown}" \
        --arg old_base_sha "${OLD_BASE_SHA:-unknown}" \
        --arg upstream_head_sha "${UPSTREAM_HEAD_SHA:-unknown}" \
        --arg upstream_commit_count "${UPSTREAM_COMMIT_COUNT:-unknown}" \
        --arg resolution "${resolution}" \
        --argjson issue_expected "${issue_expected}" \
        '{workflow:$workflow,run_url:$run_url,outcome:$outcome,reason:$reason,
          verdict:$verdict,uncertain:$uncertain,rebase_status:$rebase_status,
          fork_head_sha:$fork_head_sha,old_base_sha:$old_base_sha,
          upstream_head_sha:$upstream_head_sha,upstream_commit_count:$upstream_commit_count,
          resolution:$resolution,issue_expected:$issue_expected}')"
      jq -e 'type == "object"' >/dev/null <<<"${payload}"
      report="$(jq -n \
        --arg event_id "${event_id}" \
        --arg occurred_at "${occurred_at}" \
        --arg event "minimal-mistakes.upstream-softfork.completed" \
        --arg outcome "${outcome}" \
        --arg repository "${GITHUB_REPOSITORY}" \
        --argjson payload "${payload}" \
        '{schema:1,event_id:$event_id,occurred_at:$occurred_at,
          source:"github-actions",event:$event,outcome:$outcome,
          payload:$payload,repository:$repository}')"

      for attempt in 1 2 3; do
        status="$(curl --silent --output /dev/null --write-out '%{http_code}' \
          --max-time 10 -H 'Content-Type: application/json' \
          --data "${report}" "${NOTIFICATION_URL}" 2>/dev/null)" || status=000
        case "${status}" in
          2??) exit 0 ;;
        esac
        sleep "${attempt}"
      done

      printf 'report submission failed after 3 attempts (last HTTP status %s)\n' \
        "${status}" >&2
      exit 1

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

- A healthy run with no upstream commits submits no report and creates no issue.
- Any upstream-change outcome submits an automated report, including clean retained, rebase conflict, exact replacement, semantic replacement, and uncertainty.
- Rebase conflict, exact replacement, semantic replacement, and uncertainty also use the gh-aw issue path.
- Reports intentionally omit issue numbers because this post-step runs before gh-aw safe outputs create or update issues.
- Preflight, agent-result, or report-delivery failures are workflow failures and submit failure reports when required data is available.
- Never expose `NOTIFICATION_URL` in the agent context or logs.
- Never create or update an issue for the clean+retained case.
- Use `safe-outputs` issue creation/update only for changed-upstream outcomes that are not clean+retained.

## Final summary

Keep the final job summary small: base SHA, upstream head SHA, upstream commit count, rebase status, verdict, issue URL if created, and report result.
