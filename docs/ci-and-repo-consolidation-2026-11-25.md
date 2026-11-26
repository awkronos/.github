# CI architecture and repository consolidation

Generated from a census captured 2026-11-26T01:55:00Z, with the run histories in item 3 of
Legacy surface read in a later pass at 2026-11-26T02:17:00Z. Every number below comes from the queries in
Method, and the same data in machine-readable form is
[`org-ci-matrix.json`](../org-ci-matrix.json) at this repository's root. Each figure is injected by
the generator from the census rather than typed, so re-running the census re-writes this page.

## Why this exists

The organization has 66 repositories, 5 of them public and 61 private.
CI lives in 128 workflow files on default branches, 2,503,443 bytes in total, and not one of
them calls a reusable workflow from this repository. The reusable set landed on 2026-11-25 is
therefore unused infrastructure, and the per-repository copies it was written to replace are still
drifting on their own.

Two consequences show up in the metadata itself rather than in opinion.

First, a copy-paste taxonomy has hardened into 5 filename families that each appear in
three or more repositories, and `ci.yml` alone appears in 29 of them. The
largest single workflow file on a default branch is 297,658 bytes; the median is
9,393 bytes. Same filename, same intent, no shared implementation, 7
distinct pins of one common action. One repository holds 3 of the ten largest
files, which is what a platform repo looks like when it has absorbed the CI of the products it hosts.

Second, runner capacity is fragmented. There are 30 repository-scoped runner
registrations, 23 of them offline, next to 9
organization-scoped registrations of which 5 are online. A job that asks for a
label no online runner carries does not fail, it waits. A point sample of queue depth across ten of
the busiest repositories returned 189 runs in `queued` state and 5 in
`in_progress`, with a single repository holding 150 of the queued runs.

## Method

| Query | What it proves |
| --- | --- |
| `gh repo list awkronos --limit 100 --json name,isArchived,isPrivate,pushedAt,primaryLanguage,defaultBranchRef,description` | the population, visibility, language, staleness |
| `gh repo list awkronos --limit 100 --json name,createdAt,updatedAt,isFork,forkCount,diskUsage,stargazerCount,repositoryTopics,homepageUrl` | repository age, fork status, whether a surface is served |
| `gh api repos/awkronos/<repo>/actions/workflows?per_page=100` | every workflow the organization has registered |
| `gh api repos/awkronos/<repo>/contents/<path>` | whether that registration still has a file on the default branch |
| `gh api repos/awkronos/<repo>/actions/runners?per_page=100` and `gh api /orgs/awkronos/actions/runners` | which runners can accept a job |
| `gh api repos/awkronos/<repo>/actions/workflows/<file>/runs?per_page=100` | how a job has been ending, by run status and conclusion |
| `gh api "repos/awkronos/<repo>/actions/runs?status=queued"` and `...status=in_progress` | how many jobs are waiting right now |

Two presentation rules, both stated in the matrix as well as here.

A public artifact cannot name private repositories, so a private record's `handle` is
`sha256("awkronos/" + name)` truncated to twelve hex characters. Any member with org read access
resolves a handle by recomputing it from the live repo list; no reader of this page can. Public
records carry their names.

Runner labels outside the organization's published label set render as `custom:` plus six hex
characters. The count of jobs that no online runner can take is unchanged by the redaction, only the
host-specific spelling is gone.

## Repository classes

Each repository gets exactly one class. The rules are mechanical and the reason strings are recorded
per repository in the matrix.

| Class | Rule | Count |
| --- | --- | --- |
| active-product | serves or ships a surface: a homepage is set, or CI contains a pages or deploy job, or the repository is a named product | 21 |
| active-math-proof | primary language is Lean, proof body or proof infrastructure | 24 |
| active-infra | platform, shared library, control plane or tooling with no product surface | 13 |
| archive-candidate | two or more staleness strikes, or one strike and no live CI | 8 |

A strike is one of: the description carries a deprecation, snapshot, probe or retained marker; ten or
more days since the last push; no workflow file on the default branch; a default branch that is not
`main` or `master`; no primary language and no CI. Fork status and a missing language are recorded but
never decide a class on their own, because several public forks are deliberate upstream-tracking
libraries.

The reusable mapping is decided by the stack that a repository's current CI actually runs, not by
class alone, and each record carries its own `target_reusables` list.

## Target architecture

Five reusable workflows are callable from this repository today.

| Reusable | Covers | Call it for |
| --- | --- | --- |
| `reusable-python-ci.yml` | install, `ruff` lint, `pytest` on a hosted runner | Python products and tooling |
| `reusable-node-pnpm-ci.yml` | frozen pnpm install, typecheck, test | web front ends |
| `reusable-lean-guard-ci.yml` | structural and receipt guards on a hosted runner, deliberately no proof build | every Lean repository, every pull request |
| `reusable-swift-ci.yml` | SwiftPM build and test with a resolved-package cache | native macOS and iOS surfaces |
| `reusable-release-boundary.yml` | public-surface voice and internal-name scan | anything a reader can see |

The migration is one line per repository:

```yaml
name: ci
on:
  pull_request:
  push:
    branches: [main]
jobs:
  verify:
    uses: awkronos/.github/.github/workflows/reusable-python-ci.yml@main
    with:
      install-command: python -m pip install -e '.[dev]'
      lint-command: ruff check .
      test-command: pytest -q
```

### Two coverage gaps and one input gap

`reusable-lean-guard-ci.yml` is honest about its own boundary: it runs guards on a hosted runner and
does not build proofs. That is right for pull requests and useless as an answer to proof
compilation. In CI today, 33 repositories build with `lake` and none of them has an organization
workflow to move to, so a Lean build reusable is the highest-value addition to this repository:

```yaml
# proposed: reusable-lean-ci.yml
on:
  workflow_call:
    inputs:
      lean-version: { type: string, required: true }
      build-target: { type: string, default: "" }
      runner-labels: { type: string, default: '["self-hosted", "ci-pool"]' }
      cache-key-salt: { type: string, default: "v1" }
      timeout-minutes: { type: number, default: 90 }
```

The `runner-labels` input matters more than the build step: it is where the organization stops
discovering a pool at job time and starts declaring one. Its default should be the
`ci-pool` label set, and a repository that needs a different class of machine should say so at the
call site.

Rust sits in the same place: 22 repositories build it in CI, 12 of them alongside
Lean, and there is no reusable for it either, so each maintains its own toolchain pin and its own
cache. A `reusable-rust-ci.yml` with a toolchain input, `cargo test` plus `clippy`, and one shared
target cache retires that family.

One input gap in the Python reusable: 10 repositories install with `uv`, and the default
`install-command` covers `requirements-dev.txt` and `pyproject.toml` through pip, not uv. Those
repositories can migrate today, but only by passing `install-command` explicitly, so the default
should grow a uv branch (`uv sync --extra dev` when `uv.lock` is present) rather than leave
10 repositories carrying a private install step.

With the set that exists now, 16 repositories can replace their whole verification job with
a single caller: no `lake`, no `cargo`, not an archive candidate. The other
42 non-archive repositories keep at least one repository-local job, a proof build, a
Rust build, a deploy or a pages publish, and the 8 archive candidates retire theirs
entirely. Separately, 41 repositories are Lean-eligible, by primary language or by a
`lake` step in their current CI, and can adopt `reusable-lean-guard-ci.yml` on pull requests today.
That is the largest single adoption available and it needs no new work in this repository, which is the
honest shape of wave 0: a pilot that proves the wiring, not a big bang.

### This repository has no CI of its own

All five reusable files in `.github/.github/workflows` trigger on `workflow_call` only, and the repository has
no push or pull_request workflow. So nothing checks a change to the files that every other
repository will come to depend on: a syntax error, a renamed input or a drifted default lands silently
and breaks the callers at their next pull request, where the failure points at the caller rather than
at this repository. The cheapest fix is a self-test that calls each reusable against a fixture, which
is also the only way to learn that an input default is wrong before every caller inherits it.

## Migration waves

Wave 0, pilot and self-test. Add a self-test workflow to this repository, then add one caller to each
of: one Python product repository, one pnpm front end, one Lean proof repository. Three pull
requests, each a file replacement, each observable as a run whose job came from this repository.

Wave 1, active-product, 21 repositories. Replace install, lint and test bodies with the
Python or pnpm caller. Deploy and pages jobs stay repository-local because they carry environment and
secret context that a reusable should not.

Wave 2, active-math-proof, 24 repositories. Replace structural guard steps with
`reusable-lean-guard-ci.yml` on pull requests, and route the proof build to the org pool label set
until `reusable-lean-ci.yml` lands. The proof build is the expensive job in the organization, so it
gets one declared path with a cache key and a hard timeout, not 33 private approximations.

Wave 3, active-infra, 13 repositories. Single caller plus `reusable-release-boundary.yml`
wherever a reader can see the output.

Wave 4, archive candidates, 8 repositories. Turn read-only, delete the workflow files, and
drop the runner registrations. Archiving rather than deleting keeps links and citations working, and
2 of the candidates are public forks of upstream libraries, so for those a
read-only banner is the right ending and deletion is not.

Every record in the matrix carries `target_reusables` and `decision`, so each wave is a filter on the
matrix file rather than a list someone maintains by hand.

## Legacy surface ready for removal

Measured, grouped by the action each item needs, with the exact rows in
[`org-ci-matrix.json`](../org-ci-matrix.json) under `prune`.

1. **19 registrations have no file on the default branch.** `actions/workflows` lists each
   one, in 13 cases with state `active`, and `contents` returns 404 for the path. They sit in
   7 repositories; one registration per repository was re-queried after the census and all
   7 returned 404 again, so these are stable rather than a replication lag. 6
   of them have `probe`, `oneshot` or `canary` in the path, which is the signature of a one-shot
   experiment whose file was deleted and whose registration was left behind. Cost: the CI inventory
   overstates what is live, and badge and usage counts keep ticking on work that cannot run. Action:
   restore the file where the purpose still exists, otherwise treat the row as dead inventory.
2. **7 `disabled_manually` registrations.** 6 of them are the
   fileless rows in item 1, so the row goes when the registration is cleaned up. The remaining one
   still has a file on its default branch and is switched off, which is the honest case for deleting
   the file: a workflow that is off, unedited and still committed reads as available capacity to the
   next person who audits this organization.
3. **4 active files ask for labels no online runner carries.** Their jobs queue rather
   than fail, so three of the four run histories recorded in the matrix settle it. A scheduled stale-CI reaper in one
   private repository, firing every thirty minutes, has 165 recorded runs; in the most
   recent 100 of them 98 finished and every one of those finished with
   conclusion `cancelled`, 2 were still pending when sampled, and
   **none produced a success or a failure**, with the newest run created 2026-11-26T02:05:01Z. The tool
   meant to clear stalled CI has never itself run to a result. A GPU gate in another private repository
   has 0 recorded runs in its entire history. A deploy job in a third has
   462 runs, 51 failures and 47 cancellations in the recent
   window, so it is not silent, it is simply red. Same defect, three angles: a schedule that no
   runner can serve is not an error, and nothing reports it.
4. **23 offline repository-scoped runners.** One registration per repository,
   almost all pointing at machines that no longer check in. Action, after their jobs move to the org
   pool: `gh api -X DELETE repos/awkronos/<repo>/actions/runners/<id>`. Removing them is what turns
   queue depth around, because it stops the scheduler from offering work to machines that will never
   take it.
5. **7 scheduled files sit on repositories whose own runner pool is empty.** They fall
   back to the org pool or they stall. Either way a schedule belongs with a runner that can serve it,
   and each of these should be re-labelled to the pool or moved behind the reusable.
6. **121 of the 128 live files call `actions/checkout`, and it is pinned
   4 different ways by mutable tag and 3 different ways by commit
   SHA.** The tag spellings in use are `v4` in 37 files, `v7` in 32 files, `v5` in 13 files, `v7.0.1` in 5 files, and 35 files pin a SHA.
   Pinning by SHA is the right supply-chain default. The defect is that the SHA set is a
   per-repository accident rather than one organization decision, and a reusable is the only place in
   the organization where that decision can be made once.

## Voice and boundary

`reusable-release-boundary.yml` is the enforcement point: it scans the paths you name for internal
identifiers and fails the job on a hit. Two notes from this audit.

First, its banned-identifier list is short and hard-coded, while the public surface now includes
generated documents like this one, produced from data about repositories that are private. The
identifier list should be a single documented input with a default, so a repository never keeps its
own copy of the policy, and the scan should take the file extensions a generated page can arrive in.

Second, and this is the rule that shaped both artifacts: a public surface cannot name private
repositories. 61 of the 66 repositories here are private, so the matrix keys them by
handle, and this page counts by class instead of listing names. Names are recoverable by any member
with org read access through the documented handle rule and by no one else. Prose on the public
surface stays free of internal product codenames, host names, machine labels and em dashes, and both
files in this change were checked against that rule before they were committed.

## Acceptance

A wave is done when, for the repositories in it:

- `ci_callers_of_org_reusables` is at least one,
- `ci_self_hosted_files` has fallen to only the jobs that genuinely need the org pool,
- no active file asks for a label no online runner carries, so `jobs_no_online_runner` is empty,
- the count of registrations with no file on the default branch has stopped growing,
- the repository's own CI no longer carries an install, lint or test body that the reusable runs,
- and this repository runs a workflow on its own pull requests, so a reusable is proven here before
  it is proven in ten callers.

The census is one scripted pass over the Method commands, a few hundred read calls fanned out in
parallel batches, so the ledger can stay current on a schedule instead of being rewritten by hand each
quarter.
