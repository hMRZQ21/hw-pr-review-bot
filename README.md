# hw-pr-review-bot

A GitHub Actions bot that reviews and merges student homework pull requests, built for the [CUNY Tech Prep](https://github.com/CUNYTechPrep) data science course and running there in production.

Students submit homework as pull requests from forks of one shared course repository. This bot checks each submission, labels and explains what is wrong, and merges the ones that pass, so instructors stop triaging PRs by hand.

**Author:** Hussam Marzooq. **Contributor:** Faizan Khan.

**Where it runs:**

- [`CUNYTechPrep/2026-ds-summer-prep`](https://github.com/CUNYTechPrep/2026-ds-summer-prep/blob/main/.github/workflows/Review-HW-PRs.yml): where it debuted (summer 2026).
- [`CUNYTechPrep/ds-fall-2026`](https://github.com/CUNYTechPrep/ds-fall-2026/blob/main/.github/workflows/Review-HW-PRs.yml): the current version, running in the fall for a central repo shared across sections.

**This repo:** documented snapshots of both, copied on 2026-09-19. They sit in `workflow/` on purpose, not `.github/workflows/`, so they do not run against this repo.

- `workflow/Review-HW-PRs.yml`: the current fall version (720 lines). One example student name in a comment was replaced with a placeholder; nothing else differs from the source.
- `workflow/summer-v1/Review-HW-PRs.yml`: the original summer version (484 lines), byte-identical to the source.

## What it checks

Every open homework PR gets four checks. Each failing check maps to one label. The table describes the summer v1 checks; the fall rewrite keeps the same four labels and tightens the rules (see below).

| # | Check | Label on failure |
|---|---|---|
| 1 | No merge conflict with `main` | `bug` |
| 2 | Exactly one homework file, correctly named | `duplicate` |
| 3 | No other files touched | `invalid` |
| 4 | The notebook is actually filled in, not the blank template | `question` |

All four pass: the PR is squash-merged. Any fail: the matching labels are applied and a "Request changes" review explains each problem.

## Design notes

- **Real work detection.** Check 4 parses the `.ipynb` JSON and compares cell source text only against the template for that week. Outputs, execution counts, and metadata are ignored, because those change just from running a notebook. If the template cannot be found, the check fails closed rather than passing.
- **Feedback without spam.** Labels are synced on every run: added when a check fails, removed when it starts passing. A review comment is posted only for labels that are newly applied, so a student is not re-commented on every push.
- **Specific error messages.** The file check distinguishes "you edited the template", "right name, wrong folder", and "not recognized as a submission", because "remove those changes" is wrong advice for someone whose only mistake is a typo in the filename.
- **Two naming eras (summer v1).** One regex accepts both filename conventions in use, requires a unique per-student prefix (full name or ID) to avoid collisions in a shared repo, and tolerates OS copy suffixes such as `_copy`.
- **Safe merges.** Before merging it dismisses its own stale "changes requested" reviews (which would otherwise block a merge when reviews must be resolved), and merges with the checked commit SHA so it cannot merge a newer push it has not evaluated.
- **Scope guard.** It only acts on PRs from forks. A PR whose branch lives in the same repository (an instructor working on course content) is left untouched, whatever it changes.
- **Draft and unready PRs.** Drafts are skipped. If GitHub is still computing whether a PR is mergeable, the bot retries briefly and otherwise leaves it for the next run instead of guessing.

## What changed in the fall rewrite

The fall version is a rewrite for a repo shared across several sections, with one folder per week. It keeps the core design and the security posture below and changes the rules around them.

- **Looser, principled naming.** A Week 1 audit of 49 flagged PRs found about 26 were rejected only for punctuation or for the word after the week number, and the template and instructions had caused both. Enforcement now requires only what carries meaning: an ID that cannot collide across sections and a knowable week number. Bare initials are still rejected.
- **Week must match the folder.** A Week 3 submission filed under the Week 5 folder is flagged.
- **One week per PR.** A PR that bundles two weeks is flagged, with different wording from an accidental double submission.
- **Non-notebook files get their own message,** for example a generated `joined.csv` committed by accident.
- **Stale-PR auto-close.** A PR carrying a fail label with no new commit for 3 weeks is closed with instructions for reopening. A passing PR that is stuck for another reason is never closed.
- **Documented repo dependency.** The header records a repo setting the merges depend on and how it was diagnosed, so the next maintainer checks it first.

## Threat model

The workflow uses `pull_request_target`, the trigger security write-ups warn about. Fork PRs get only a read-only token under a plain `pull_request` trigger, so `pull_request_target` is the only way for the bot to label, review, and merge them. It is dangerous when a workflow also checks out and runs the fork's code with that elevated token. This one does not:

- **No checkout step exists.** The workflow has a single step.
- **Fork content is only read through the API and parsed as JSON.** It is never executed. The rest is metadata calls: mergeable state, file list, labels.
- **A PR cannot change the workflow's behavior.** The copy of the workflow in the base repository is the one that runs, whatever a PR modifies.
- **The one third-party action is pinned to a full commit SHA** (`actions/github-script`, v7.1.0), not a floating tag.

## Triggers and rollout

- **`pull_request_target`** (opened, synchronize, reopened): reacts immediately when a PR opens or a student pushes a fix.
- **Hourly schedule:** a backstop, not the main path. It catches PRs whose mergeable state was still computing, and PRs that existed before the workflow did.
- **Manual run with a dry-run switch** (`workflow_dispatch`): logs exactly what it would do for every open PR without touching a label, review, or merge. Compare that log against PRs you have already triaged by hand before turning it live. Dry run applies to manual runs only.

## Impact

- Reclaims about 3 hours of instructor time per week (reported by the author).
- Projected to save 8 hours per week across a 5-person instruction team in Fall 2026. This is a projection, not a measurement.

## Reusing it

1. Copy `workflow/Review-HW-PRs.yml` to `.github/workflows/` in your course repository.
2. Adjust the tunables at the top of the script: `NAME_RE`, `WEEK_FOLDER_RE`, `TEMPLATE_PREFIX`, `MERGE_METHOD`, and `STALE_WEEKS`. Check the folder layout described in the file header against your repository.
3. Run it once manually with **Dry run** checked and read the log before you let it act.

The workflow needs `pull-requests`, `contents`, and `issues` write permissions.

## License

MIT. See [LICENSE](LICENSE).
