<!-- Template for the default scenario: GitHub default branch is `master`, behind Bioconductor, no `devel` branch. Adapt the sync section for other scenarios (see SKILL.md). -->

Hello <pkg> maintainers,

<pkg> was **deprecated** on Bioconductor (commit `<sha>`, "<deprecation commit subject>", <Month YYYY>) because the maintainer did not respond to build failures, which puts it on the path to removal under the [Bioconductor package end-of-life policy](https://contributions.bioconductor.org/package-end-of-life-policy.html#orphaned-packages).

The package has since been **rescued**: the deprecation was reverted and fixes were committed directly to the Bioconductor git repository (`git.bioconductor.org/packages/<pkg>`). These rescue commits are included in the Bioconductor 3.XX release (`RELEASE_3_XX`, <pkg> <release version>) and in `devel` (<devel version>):

- <rescue commit subject>
- <rescue commit subject>

This GitHub repository (last commit `<sha>`, <Month YYYY>) is now <N> commits behind Bioconductor. Please update it so that it matches the Bioconductor linear history **before making any new changes**. Otherwise, future pushes to Bioconductor will fail or create divergent histories.

## Use `devel` as the default branch

Bioconductor uses `devel` as the name of its development branch, and we recommend that the GitHub repository do the same instead of `master`. With matching branch names, pushing to and pulling from both remotes is simpler and less error-prone. See the Bioconductor blog post [Transition to `devel` as the default branch](https://blog.bioconductor.org/posts/2023-03-01-transition-to-devel/) for more information.

The easiest way to rename it is on GitHub: **Settings → Branches → Default branch → rename `master` to `devel`**. GitHub keeps redirects for the old name and retargets open pull requests. Or, with the GitHub CLI:

```sh
gh api -X POST repos/<owner>/<pkg>/branches/master/rename -f new_name=devel
```

## How to sync this repository with Bioconductor

Before you start, make sure your SSH key is registered with the [Bioconductor Git credentials app](https://git.bioconductor.org/BiocCredentials/) if you plan to push to Bioconductor.

```sh
# 1. Clone (or use your existing clone of) the GitHub repository
git clone git@github.com:<owner>/<pkg>.git
cd <pkg>

# 2. If you have an existing clone, update it after renaming master to devel on GitHub
git branch -m master devel
git fetch origin
git branch -u origin/devel devel
git remote set-head origin -a

# 3. Add the Bioconductor remote (skip if it already exists)
git remote add upstream git@git.bioconductor.org:packages/<pkg>.git
git fetch --all

# 4. Reset your devel branch to the Bioconductor devel branch
git checkout devel
git reset --hard upstream/devel

# 5. Force update GitHub so it matches Bioconductor exactly
git push --force origin devel

# 6. (Optional) Mirror the release branch as well
git push origin upstream/RELEASE_3_XX:refs/heads/RELEASE_3_XX
```

> [!NOTE]
> Because the GitHub branch has no commits that are missing from Bioconductor, `git merge --ff-only upstream/devel` followed by a normal `git push origin devel` gives the same result. If you have local or unpushed work, rebase it onto `upstream/devel` instead of using `reset --hard`.

From then on, keep both remotes in sync:

```sh
git pull upstream devel   # pick up changes from Bioconductor (e.g. version bumps)
# ...make your changes and commit...
git push upstream devel   # push to Bioconductor
git push origin devel     # push to GitHub
```

For more details, see the Bioconductor guides on [syncing an existing GitHub repository](https://contributions.bioconductor.org/git-version-control.html#sync-existing-repositories) and the [package end-of-life policy](https://contributions.bioconductor.org/package-end-of-life-policy.html#orphaned-packages).

Please also keep an eye on the [Bioconductor build reports](https://bioconductor.org/checkResults/devel/bioc-LATEST/<pkg>/) and the email address listed as maintainer in `DESCRIPTION`, so that build problems can be fixed before the package is deprecated again. If you no longer wish to maintain <pkg>, please let us know here or on the [bioc-devel mailing list](https://stat.ethz.ch/mailman/listinfo/bioc-devel).

Thank you!
