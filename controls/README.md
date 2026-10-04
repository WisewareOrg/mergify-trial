# Merge-path controls — reference implementation

This repository is the proven reference for how a change reaches `main` in an enrolled WisewareOrg
repository. Every control below was exercised here against live GitHub, Mergify and Renovate; the
**Evidence** column names the pull request that shows it. The repository is deleted once wise-ci
carries these controls as its standard; nothing here is a dependency of any other repository.

## The model

- **The commit on `main` is the exact commit that was verified.** Mergify fast-forwards `main` to a
  pull request's head SHA; it never writes a new commit at landing.
- **One commit per pull request.** A multi-commit pull request is squashed on its own branch before it
  is queued, so every commit on `main` is a verified configuration.
- **Serial.** One pull request is checked at a time, in place, on the pull request itself.
- **Mergify is the only way into `main`.** A direct push and GitHub's merge button are both refused.
  Break-glass is an organization admin's explicit `--admin` merge, and it still requires every check.
- **Fully automatic for Renovate.** Renovate opens pull requests; Mergify rebases, verifies and lands
  them, many per Renovate run.

## Controls

| ID | Control | Lives in | Evidence |
|---|---|---|---|
| MC-1 | Ruleset *only Mergify updates main*: `pull_request` and `update` (restrict updates). Bypass: the Mergify app (`always`) and the organization-admin role (`pull_request` only — merges, never pushes). | [`rulesets/only-mergify-updates-main.json`](rulesets/only-mergify-updates-main.json) | PR 5 (*only Mergify may land*): direct push refused, *Cannot update this protected ref*. PR 6 (*invalid config*): merge button refused with every required check green. |
| MC-2 | Ruleset *main integrity*: `deletion`, `non_fast_forward`, `required_linear_history`, `required_status_checks`. **No bypass of any kind.** The required-check list is the one per-repository part of the model. | [`rulesets/main-integrity.json`](rulesets/main-integrity.json) | First probe: a direct push of an unchecked commit refused, *2 of 2 required status checks are expected*. PR 6: `--admin` refused while `bench` was pending. |
| MC-3 | Mergify app installed on the repository, with **Merge Queue**, **Merge Protections** and **Workflow Automation** each enabled in the Mergify dashboard. Installing the app alone does nothing. | GitHub app installation; Mergify dashboard | PR 1 (*three-commit PR*): no queueing, squash or auto-queue until each product was enabled. |
| MC-4 | Standard `.mergify.yml`: fast-forward, `max_parallel_checks: 1`, `batch_size: 1`, rebase as `update_bot_account`, queue conditions `#commits = 1` and `-draft`, **no `merge_conditions`**. | [`../.mergify.yml`](../.mergify.yml) | PR 1, PR 8 to PR 11 (*serial queue*): landed SHA equals head SHA. PR 3 (*draft PR 4*): any `merge_conditions` moves checking to a draft pull request with a *Merge of #N* merge commit. |
| MC-5 | Mergify reads the rulesets' required checks itself, injected as queue conditions. No check is named in `.mergify.yml`, so the file is identical in every repository. | `.mergify.yml` (absence of check names) | PR 1: the queue checklist lists `ci` and `bench` *from GitHub repository ruleset rule `main integrity`*. |
| MC-6 | Auto-queue: every non-draft pull request to `main`, except a Renovate pull request without the `automerge` label (major updates wait for a human). | `.mergify.yml` `merge_protections_settings` | PR 19 to PR 22 (*Renovate batch*) queued; PR 23 to PR 25 (majors) not queued until labelled. |
| MC-7 | Squash a multi-commit pull request on its branch (`all-commits` message), acting as the `bot_account` user — required for a Renovate pull request, whose app author Mergify cannot impersonate. | `.mergify.yml` `pull_request_rules` | PR 1 (squash as author). PR 21 (*setup-go 6.5 fix*): squash refused without `bot_account`, *Mergify can't impersonate a bot from another GitHub App*. |
| MC-8 | Hand-off: a Renovate pull request with more than one commit (someone else committed to it) gets the `stop-updating` label; Renovate then leaves it alone. Renovate always writes exactly one commit per branch. | `.mergify.yml` `pull_request_rules` | PR 24 (*setup-go v7 fix*): Renovate log *Branch updating is skipped because stopUpdatingLabel is present*. PR 25 (*setup-python v7 fix*): labelled, squashed, landed. |
| MC-9 | Renovate configuration for the model: `platformAutomerge: false`; auto-merge update types labelled `automerge` instead of merged; `prCreation: immediate`; `internalChecksFilter: strict`; `rebaseWhen: conflicted`; `gitIgnoredAuthors` lists the `update_bot_account` user's ID-form noreply address. | [`../renovate.json`](../renovate.json) (belongs in the wise-renovate preset) | See [Renovate findings](#renovate-findings). |
| MC-10 | A check that runs only after a pull request is queued — a hardware bench — posts its context as **pending** on every pull request head, and the runner verifies *candidates*: any pull request whose other checks are green and whose bench is pending, then again on the rebased head while it carries the `mergify-checking` label. A dummy pass on the pull request would satisfy the required check on the very commit that lands. | [`../.github/workflows/ci.yml`](../.github/workflows/ci.yml) `bench-pending` job; runner side is the consuming repository's | PR 8 to PR 11 and PR 19 to PR 22: candidate bench, then in-queue bench on the rebased SHA. |
| MC-11 | `.mergify.yml` is validated by **Mergify's own validator** — the *Configuration changed* check, which also blocks the change from the queue — not only by the published JSON schema. Condition strings are free text to the schema. | Mergify *Configuration changed* check | PR 29 (*hand-off rule*): `-commits[*].author = …` passed `check-jsonschema` and failed Mergify, *Invalid operator*. |
| MC-12 | Mergify's own checks are **not** required checks. With an invalid configuration on `main` they report failure or nothing, which would block the break-glass merge that fixes it. | MC-2 required-check list | PR 6 and PR 7 (*break-glass drill*): invalid config landed by `--admin`; queue dead; fix landed by `--admin` while *Mergify Merge Queue* reported failure. |

## Renovate findings

- **`not-pending` never opens a pull request** when CI runs only on `pull_request` events and
  `minimumReleaseAge` is set: the bare branch carries no external check, and the `prNotPendingHours`
  fallback is disabled by a non-zero release age (Renovate's `prCreation` documentation warns of
  exactly this). `prCreation: immediate` with `internalChecksFilter: strict` keeps the release-age
  guarantee — a version younger than the minimum age gets no branch at all.
- **Renovate treats a branch as edited** when any commit's author *or committer* email is neither its
  own nor in `gitIgnoredAuthors`. Mergify's rebase commits as the `update_bot_account` user, so that
  user's address is ignored; the hand-off signal is the `stop-updating` label (MC-8), not an email.
- **Mergify's `commits[*]` conditions mean *all* commits**, and a negated `commits[*]` condition is an
  invalid operator, so *any commit not by Renovate* is expressed as `#commits > 1` on a Renovate pull
  request.

## Throughput

One Renovate run opened four auto-merge pull requests; Mergify landed three in about three minutes
(PR 22, PR 19, PR 20), each rebased onto the previous landing and verified on its own head, and
dequeued the fourth (PR 21) on a failed in-queue bench. Strict up-to-date protection without a queue
lands one Renovate update per Renovate run.

## Enrolling a repository

1. Install the Mergify app on the repository and enable Merge Queue, Merge Protections and Workflow
   Automation (MC-3).
2. Add the standard `.mergify.yml` (MC-4 to MC-8).
3. Create both rulesets (MC-1, MC-2); MC-2's required-check list is the repository's own CI.
4. Remove GitHub's native `merge_queue` rule and any `merge_group` workflow triggers.
5. Extend the wise-renovate preset carrying MC-9, and list the repository in its `config.json`.
6. Give any post-queue check the pending-status pattern (MC-10).

## Known gaps

- **The controls are not under configuration management.** Rulesets, dashboard toggles and bypass
  lists are platform settings; a silent change removes change control without a record in any
  repository. A check that reads each repository's live settings and compares them with this standard
  is required.
- **Verification evidence expires.** Actions logs are deleted after the retention period; the check
  conclusions per SHA remain, the detail does not.
- **Author attribution on a hand-fixed Renovate pull request.** The squash keeps Renovate as author;
  the fix is recorded as a line of the commit message and in the pull request.
- **Independence.** No reviewer independent of the author is enforced.

## Trial scaffolding — not part of the standard

`case*.txt`, `pr24-fix.txt`, `pr25-fix.txt` and `.github/workflows/deps.yml` (outdated pins for the
Renovate batch) exist only to drive the trial. The `bench` context stands in for a hardware bench and
was posted by a script playing the build host.
