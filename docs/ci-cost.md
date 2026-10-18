# CI cost audit: helix-newco-agent

Fleet context (from the CI-cost dispatch, not read from a billing API here): GitHub Actions is at
**$180.82 of a $200 monthly cap** that resets **1 Nov**; fleet target **<= $1.30/day**. This repo is
**private**, so hosted-runner minutes count against that cap.

Audited ref: `main` @ `82ff35b912e4a08c435d29278631c95445adf658` (2026-10-18).

## Audit table

| Workflow | Triggers (before) | Jobs / runner | Cost drivers (evidence) | Change in this PR |
|---|---|---|---|---|
| `ci.yml` ("CI") | `push` to `main`, `pull_request` (all), `workflow_dispatch` | 7 job runs per trigger, all `ubuntu-latest`: `quality` (15 min cap), `platform` matrix web/ios/android (3 x 20 min cap, Expo export on Linux, not a real iOS build), `browser` (Playwright Chromium + OS deps), `browser-container` (Docker), `computer-container` (Docker build) | Six of the seven runs do a `pnpm install` with **no cache**. Two Docker builds and a Playwright browser install run per trigger. **No path filters**, so docs-only changes (the repo has 8 files under `docs/` plus top-level `.md`) run all 7. Run history: **this workflow has never run here**. The repo's only 4 Actions runs are Dependabot security-update jobs (event `dynamic`, all `failure`, ~1.5 min each, 2026-09-24 and 2026-10-17). | `paths-ignore` for `**/*.md` and `docs/**`. Concurrency keyed on PR number (was `github.ref`). `cache: pnpm` on every job that installs. Draft PRs skip the 4 heavy jobs (`platform`, `browser`, `browser-container`, `computer-container`); `quality` still runs on drafts. `ready_for_review` added. |

Dollar amounts: **Unknown - needs owner input** (no billing API access from this session).
Whether Dependabot update jobs bill against the Actions cap: **Unknown - needs owner input**.

## Is this an unused upstream copy?

The history (merges from external contributors, `jerel/*` and `tyler/*` branches) suggests this repo is a copy of
the upstream OpenMuse project, and `ci.yml` has never run here. GitHub does not flag it as a fork.
**If the repo is not actively developed here, the cheapest option is to disable Actions in
Settings > Actions** (owner-only, not done in this PR). That would also stop the failing
Dependabot update runs.

## Required checks

`main` reports `protected: false` (branches API, 2026-10-18), so no job is a required check through
branch protection. Rulesets: **Unknown - needs owner input**. If a ruleset requires these jobs, note that jobs
skipped by `if:` report success, and docs-only PRs skipped by `paths-ignore` report no status.

## Cron

None. No `on.schedule`; nothing to remove.

## Runners, caching, artifacts

- Runners: `ubuntu-latest` only. The `ios` matrix leg is an Expo JS export on Linux, so no macOS is needed.
- Caching: `actions/setup-node` `cache: pnpm` added (pnpm/action-setup now runs first so the store
  path resolves). Not added, and left as follow-ups: Docker layer cache for the two container jobs, and
  a Playwright browser cache.
- Artifacts: none uploaded, so no retention setting is needed.

## Conventions (copy to new workflows)

1. Concurrency: `group: ${{ github.workflow }}-${{ github.event.pull_request.number || github.ref }}`,
   `cancel-in-progress: true`. Release, publish and sequential workflows use `cancel-in-progress: false`.
2. `paths-ignore: ["**/*.md", "docs/**"]` on build/test triggers.
3. Skip draft PRs on expensive jobs (`if: github.event.pull_request.draft == false`) unless the job is a
   required check, and include `ready_for_review` in the `pull_request` types.
4. `ubuntu-latest` only, unless a real iOS/macOS build needs macOS.
5. Cache dependencies (`cache: pnpm`); `retention-days: 3`-`7` on any uploaded artifact.
6. No cron without a written justification here.
7. `[skip ci]` in commit messages for docs/workflow-policy-only changes.
