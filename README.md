# rescue-sync-issue

A [Claude Code](https://claude.com/claude-code) skill for the
[bioc-package-rescue](https://github.com/bioc-package-rescue) effort.

When a Bioconductor package is deprecated for an unresponsive maintainer and
then rescued with commits pushed directly to `git.bioconductor.org`, the
maintainer's GitHub repository falls behind the Bioconductor history. This
skill opens an issue on that repository that:

- explains that the package was deprecated and rescued, linking the
  [package end-of-life policy](https://contributions.bioconductor.org/package-end-of-life-policy.html#orphaned-packages)
- lists the rescue commits and the release and devel versions that contain them
- recommends `devel` as the default branch
  ([why](https://blog.bioconductor.org/posts/2023-03-01-transition-to-devel/))
- gives step-by-step commands to force-sync GitHub with Bioconductor, adapted
  to the repository's actual state (behind only, diverged, existing `devel`
  branch, ...)

## Examples

- [aertslab/RcisTarget#45](https://github.com/aertslab/RcisTarget/issues/45):
  `master` behind Bioconductor, no `devel` branch
- [alyssafrazee/ballgown#193](https://github.com/alyssafrazee/ballgown/issues/193):
  `master` diverged (patch-equivalent commits), stale `devel` branch already present

## Install

As a Claude Code plugin:

```text
/plugin marketplace add bioc-package-rescue/rescue-sync-issue
/plugin install rescue-sync-issue@bioc-package-rescue
```

Or copy `skills/rescue-sync-issue/` into `~/.claude/skills/`.

## Use

From a local clone of the rescued package (with a remote pointing at
`git.bioconductor.org`), ask Claude something like:

> Create an issue on the package's GitHub repository telling the maintainer
> it was rescued and how to sync with Bioconductor.

Requires the [GitHub CLI](https://cli.github.com/) (`gh`), authenticated.
