---
name: rescue-sync-issue
description: Open a GitHub issue on the upstream repository of a deprecated-then-rescued Bioconductor package, telling the maintainer it was rescued and how to force-sync their GitHub repository (and switch to a `devel` default branch) so it matches the Bioconductor linear history. Use when asked to notify a maintainer about rescue commits or to "do the same" rescue notification for another package.
metadata:
  version: "1.0.0"
  category: r-packages
  author: bioc-package-rescue
  tags: bioconductor, rescue, deprecation, git, github, maintainer
---

# rescue-sync-issue

A rescued package was deprecated on Bioconductor (usually a "Deprecate(d)
unresponsive" commit), then rescued: the deprecation was reverted and fixes
were pushed directly to `git.bioconductor.org`. The maintainer's GitHub
repository is now behind, and often diverged from, Bioconductor. This skill
writes an issue on that repository telling the maintainer what happened and
exactly how to sync.

## Usage

- "Create an issue on the maintainer's GitHub repository saying the package
  was rescued and how to sync it with Bioconductor"
- "Tell the maintainer to force update GitHub to match the Bioconductor
  history"
- "Do the same rescue notification for ballgown"

## Prerequisites

- A local clone of the rescued package with a remote (usually `origin`)
  pointing at `git@git.bioconductor.org:packages/<pkg>`.
- The GitHub CLI (`gh`), authenticated with permission to open issues.
- The package's `DESCRIPTION` lists a GitHub repository in `BugReports` or
  `URL`.

## Process

### 1. Gather facts (never guess them)

Read them from the Bioconductor remote:

```sh
git fetch origin
grep -E '^(Package|Version|BugReports|URL)' DESCRIPTION
# latest RELEASE_3_xx branch
git ls-remote origin | grep -E 'RELEASE_|devel'
git show origin/RELEASE_3_XX:DESCRIPTION | grep Version
git show origin/devel:DESCRIPTION | grep Version
# deprecation and revert commits
git log --oneline -i --grep=deprecat origin/devel
# rescue commits
git log --format='%h %ad %an %s' --date=short -30 origin/devel
```

The issue goes to the GitHub repository in `BugReports` (or `URL`). Confirm
it with `gh repo view OWNER/REPO --json defaultBranchRef,isArchived,hasIssuesEnabled`.
Stop and tell the user if the repository is archived, has issues disabled, or
isn't on GitHub.

### 2. Check the GitHub repository's real state

The sync steps depend on this, so check it every time, for the default branch
and any existing `devel` branch:

```sh
gh api repos/OWNER/REPO/branches --jq '.[].name'
git fetch -q https://github.com/OWNER/REPO.git BRANCH:refs/tmp/gh-BRANCH
# behind Bioconductor
git rev-list --count refs/tmp/gh-BRANCH..origin/devel
# commits not in Bioconductor
git rev-list --count origin/devel..refs/tmp/gh-BRANCH
# '-' = already in Bioconductor (patch-equivalent), '+' = not
git cherry origin/devel refs/tmp/gh-BRANCH
git diff --stat origin/devel refs/tmp/gh-BRANCH
git update-ref -d refs/tmp/gh-BRANCH
```

Decide which scenario applies:

| GitHub state | Instructions |
| --- | --- |
| Default branch is an ancestor of Bioc `devel` (0 commits ahead) | Fast-forward. Still give the force-push steps, with a note that `merge --ff-only` + plain push gives the same result. |
| Diverged, but every extra commit is patch-equivalent (`git cherry` shows only `-`) or a merge of those | Force push needed. Say explicitly that no content is lost and name the PR/commits that are already in Bioconductor. |
| Diverged with real content not in Bioconductor (`+` lines) | **Stop and ask the user** before writing the issue. The maintainer must rebase that work onto `upstream/devel` and push it to Bioconductor first. |
| Default is `master`, no `devel` branch | Rename `master` → `devel` on GitHub (Settings → Branches, or the `branches/master/rename` API), then update local clones. |
| A stale `devel` branch already exists | Don't rename. Push Bioc `devel` to it, make it the default branch, then delete or force-update `master`. |

Also note files that will vanish from GitHub after the sync (present on
GitHub, removed in Bioconductor) and mention them to the user, not
necessarily in the issue.

### 3. Write the issue

Fill in [references/issue-template.md](references/issue-template.md) and adapt
its "How to sync" section to the scenario above. Requirements:

- Say the package was **deprecated** (cite the commit and month) and has been
  **rescued**; link
  <https://contributions.bioconductor.org/package-end-of-life-policy.html#orphaned-packages>.
- List the rescue commits (subjects, not hashes) and the `RELEASE_3_xx` and
  `devel` versions that contain them.
- Give the exact ahead/behind counts and the last GitHub commit.
- Recommend `devel` as the default branch and link
  <https://blog.bioconductor.org/posts/2023-03-01-transition-to-devel/>.
- Give copy-pasteable commands with the real owner, repository and package
  names.
- Never write a bare `@name` (GitHub notifies it). Referring to an issue or
  PR as `#N` is fine but cross-links it.
- Keep each paragraph on one line: GitHub renders line breaks in issue bodies.

Title: `<pkg> was deprecated and rescued on Bioconductor: please sync GitHub
with the Bioconductor history`

### 4. Create the issue

Write the body to a temporary file and open the issue with
`gh issue create -R OWNER/REPO --title "..." --body-file BODY.md`.

To change it later, edit the body in place with
`gh issue edit N -R OWNER/REPO --body-file BODY.md`. Editing sends no
notification; tell the user so and offer to add a comment if the change
matters to someone who already read it.

## Output Format

- The URL of the new issue.
- Anything about the GitHub repository's state that made the instructions
  differ from the default: divergence (and why no content is lost), an
  existing `devel` branch, files that will be removed by the sync.

## Examples

**User**: "Create an issue at aertslab/RcisTarget telling the maintainer the
package was rescued and how to force update GitHub to match Bioconductor."

**Result**: [aertslab/RcisTarget#45](https://github.com/aertslab/RcisTarget/issues/45).
GitHub `master` was 32 commits behind Bioconductor with nothing extra, and
there was no `devel` branch, so the issue says to rename `master` to `devel`
and notes that a fast-forward works too.

**User**: "Can you do the same for ballgown?"

**Result**: [alyssafrazee/ballgown#193](https://github.com/alyssafrazee/ballgown/issues/193).
GitHub `master` had diverged through a merged PR whose two commits were
already in Bioconductor under other hashes, and a stale 2016 `devel` branch
existed, so the issue says to push to `devel`, make it the default, and
retire `master` with a force push.

## Notes

- Bioconductor's sync guide:
  <https://contributions.bioconductor.org/git-version-control.html#sync-existing-repositories>
- If the GitHub repository has work that isn't in Bioconductor, sync must
  happen in the other direction first; don't tell the maintainer to
  force push over it.
