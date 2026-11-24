---
name: rescue-sync-issue
description: Open a GitHub issue on the upstream repository of a deprecated-then-rescued Bioconductor package, telling the maintainer it was rescued and how to force-sync their GitHub repository (and switch to a `devel` default branch) so it matches the Bioconductor linear history. Use when asked to notify a maintainer about rescue commits or to "do the same" rescue notification for another package.
---

# Rescue sync issue

A rescued package was deprecated on Bioconductor (usually a "Deprecate(d)
unresponsive" commit), then rescued: the deprecation was reverted and fixes
were pushed directly to `git.bioconductor.org`. The maintainer's GitHub
repository is now behind, and often diverged from, Bioconductor. This skill
writes an issue on that repository telling the maintainer what happened and
exactly how to sync.

Run it from a local clone of the package whose `origin` (or another remote)
points at `git@git.bioconductor.org:packages/<pkg>`.

## 1. Gather facts (never guess them)

```sh
git fetch origin
git remote -v
grep -E '^(Package|Version|BugReports|URL)' DESCRIPTION
git ls-remote origin | grep -E 'RELEASE_|devel'          # latest RELEASE_3_xx
git show origin/RELEASE_3_XX:DESCRIPTION | grep Version   # release version
git show origin/devel:DESCRIPTION | grep Version          # devel version
git log --oneline -i --grep=deprecat origin/devel         # deprecation + revert commits
git log --format='%h %ad %an %s' --date=short -30 origin/devel  # rescue commits
```

The issue goes to the GitHub repository in `BugReports` (or `URL`). Confirm with
`gh repo view OWNER/REPO --json defaultBranchRef,isArchived,hasIssuesEnabled`.
Stop and tell the user if the repository is archived, has issues disabled, or
isn't on GitHub.

## 2. Check the GitHub repository's real state

The steps in the issue depend on this, so check it every time:

```sh
gh api repos/OWNER/REPO/branches --jq '.[].name'
git fetch -q https://github.com/OWNER/REPO.git BRANCH:refs/tmp/gh-BRANCH   # each relevant branch
git rev-list --count refs/tmp/gh-BRANCH..origin/devel   # behind Bioconductor
git rev-list --count origin/devel..refs/tmp/gh-BRANCH   # commits not in Bioconductor
git cherry origin/devel refs/tmp/gh-BRANCH              # '-' = already in Bioc (patch-equivalent)
git diff --stat origin/devel refs/tmp/gh-BRANCH
git update-ref -d refs/tmp/gh-BRANCH                    # clean up
```

Decide which scenario applies:

| GitHub state | Instructions |
|---|---|
| Default branch is an ancestor of Bioc `devel` (0 commits ahead) | Fast-forward. Still give the force-push steps, with a note that `merge --ff-only` + plain push gives the same result. |
| Diverged, but every extra commit is patch-equivalent (`git cherry` shows only `-`) or a merge of those | Force push needed. Say explicitly that no content is lost and name the PR/commits that are already in Bioconductor. |
| Diverged with real content not in Bioconductor (`+` lines) | **Stop and ask the user** before writing the issue. The maintainer must rebase that work onto `upstream/devel` and push it to Bioconductor first. |
| Default is `master`, no `devel` branch | Rename `master` → `devel` on GitHub (Settings → Branches, or `gh api -X POST repos/OWNER/REPO/branches/master/rename -f new_name=devel`), then update local clones. |
| A stale `devel` branch already exists | Don't rename. Push Bioc `devel` to it, make it the default branch, then delete or force-update `master`. |

Also note files that will vanish from GitHub after the sync (present on GitHub,
removed in Bioconductor) and mention them to the user, not necessarily in the
issue.

## 3. Write the issue

Fill in `references/issue-template.md` and adapt the "How to sync" section to
the scenario above. Requirements:

- Say the package was **deprecated** (cite the commit and month) and has been
  **rescued**; link
  <https://contributions.bioconductor.org/package-end-of-life-policy.html#orphaned-packages>.
- List the rescue commits (subjects, not hashes) and the `RELEASE_3_xx` and
  `devel` versions that contain them.
- Give the exact ahead/behind counts and the last GitHub commit.
- Recommend `devel` as the default branch and link
  <https://blog.bioconductor.org/posts/2023-03-01-transition-to-devel/>.
- Give copy-pasteable commands with the real owner, repo and package names.
- Never write a bare `@name` (GitHub notifies it). Referring to an issue or
  PR as `#N` is fine but cross-links it.

Title: `<pkg> was deprecated and rescued on Bioconductor: please sync GitHub with the Bioconductor history`

## 4. Create it

Write the body to a temporary file and run:

```sh
gh issue create -R OWNER/REPO --title "..." --body-file BODY.md
```

To change it later, edit the body in place (`gh issue edit N -R OWNER/REPO
--body-file BODY.md`). Editing sends no notification; tell the user so and
offer to add a comment if the change matters to someone who already read it.

Report the issue URL to the user, plus anything about the repository's state
that made the instructions differ from the default (divergence, existing
`devel` branch, files that will be removed).
