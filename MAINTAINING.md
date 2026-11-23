# Maintaining the BDCHM Explorer

How to keep JHU BIDS's copy of the Explorer current with the upstream BDCHM
schema: one-time setup, then what to do each time a schema update arrives.

## What happens without you

A GitHub Action ([schema-sync.yml](.github/workflows/schema-sync.yml)) checks
the upstream schema
([RTIInternational/NHLBI-BDC-DMC-HM](https://github.com/RTIInternational/NHLBI-BDC-DMC-HM))
every day at 07:00 UTC. When upstream has changed, it downloads the new schema,
regenerates the app's data file, and opens a pull request titled **Sync
upstream BDCHM schema**. [CODEOWNERS](.github/CODEOWNERS) makes GitHub request
your review, so you get a notification.

That's all it does. Nothing merges on its own, merging doesn't update the live
site, and nothing warns you if the PR sits there. The live site
(https://jhu-bids.github.io/bdchm-explorer/) changes only when someone runs
`npm run deploy`.

If the repo has no commits for 60 days, GitHub switches the daily run off. The
Actions tab then shows a banner on *Schema sync*; click **Enable workflow**.

## One-time setup

You need Git, Node 24, GNU Make, the GitHub CLI (`gh`) and uv. uv runs the
Python scripts and installs the Python version they need, so you don't install
Python yourself.

### Windows (Git Bash)

In PowerShell:

```
winget install --id GitHub.cli -e
winget install --id astral-sh.uv -e
winget install --id ezwinports.make -e
```

For Node, download the **Node 24** Windows installer from
https://nodejs.org/en/download (choose v24 in the version menu). Use 24, not a
newer major version: on Node 26, 41 of the tests fail.

Close and reopen Git Bash, then check that each one is found:

```
node --version    # v24.x
make --version
gh --version
uv --version
```

### macOS

`make` comes with the Xcode command-line tools (`xcode-select --install` if
it's missing). Then:

```
brew install gh uv
```

For Node 24: with nvm, `nvm install` in the repo directory reads
[.nvmrc](.nvmrc); otherwise use the v24 installer from nodejs.org.

### Both

Sign in to GitHub from the command line. Choose GitHub.com, and if it asks
about SSO for `jhu-bids`, authorize it:

```
gh auth login
```

Clone the repo and install:

```
git clone https://github.com/jhu-bids/bdchm-explorer.git
cd bdchm-explorer
npm ci
```

Check that it runs: `npm run dev`, then open
http://localhost:5173/bdchm-explorer/.

## When a sync PR arrives

All commands run from the repo directory in Git Bash (Windows) or Terminal
(macOS).

### 1. Run the review

```
git checkout main
git pull
make sync-review
```

It prints four parts:

- **Status**: the open PR and which files it changes.
- **Audit**: what changed in the schema that the PR's diff can't show you
  plainly. Sections marked `[!]` need a look; `[.]` sections are
  informational or empty.
- **Tests and build**: if either fails, it stops there.
- **What to do next**, based on the above.

The first run takes a few minutes longer while uv sets up Python.

### 2. Act on what the audit flags

| Audit section | What it means | What to do |
| --- | --- | --- |
| Classes added / removed | Upstream added or dropped a class. A new class that isn't in a category doesn't appear anywhere in the app. | Add it to a category in [entityCategories.ts](src/config/entityCategories.ts). |
| Slot sites added / removed | A field appeared on or disappeared from a class. Usually ordinary schema change. | Open those classes in the app (step 3) and check they still look right. |
| Hand-curated override sets vs. schema | A field named in [containmentGraph.ts](src/models/containmentGraph.ts) was renamed or removed. | Update the set in that file to the new name. |
| Range / cardinality changes, or classes that became ownership roots | What owns what in the diagram may have changed. | A judgment call; see [OWNERSHIP_CLASSIFICATION.md](docs/OWNERSHIP_CLASSIFICATION.md). Ask Siggie if unsure. |
| Classes that lost every inbound reference | A class now appears with nothing pointing at it. | Usually a field's range widened to `Entity`. Decide whether that's acceptable. |

If nothing needs a code change, go to step 3.

If something does, make the change on the PR's branch and merge it the same
day, since the next daily run rewrites that branch if upstream has moved again:

```
make sync-checkout
# edit, then:
git commit -am "Categorize <NewClass>"
git push origin HEAD
make sync-review
```

### 3. Look at the app

The tests don't catch everything: an upstream rename once left a class with no
connections in the diagram while every test passed. With the PR's branch
checked out (`make sync-checkout`), run `npm run dev` and look at the classes
the audit named.

### 4. Merge

Use the **Merge** button on the PR page, or:

```
gh pr merge schema-sync/upstream-update --squash --delete-branch
```

### 5. Deploy

```
git checkout main
git pull
npm run deploy
```

This builds the site and pushes it to the `gh-pages` branch, which GitHub
Pages serves. The live site updates within a minute or two; reload it to
check.

## Updating by hand

To pull the latest upstream schema without waiting for the daily run, either
click **Run workflow** on *Schema sync* in the Actions tab (it opens the PR as
usual), or run `make sync-manual` locally, which updates the files in your
working copy for you to commit. `make sync-check` only reports whether
upstream has moved.

## More detail

- [README.md](README.md): what the Explorer shows and how it's built.
- [OWNERSHIP_CLASSIFICATION.md](docs/OWNERSHIP_CLASSIFICATION.md): the
  ownership rules behind the diagram.
- `make help` lists every make target.

This copy was made from
[Sigfried/dynamic-model-var-docs](https://github.com/Sigfried/dynamic-model-var-docs),
which Sigfried Gold continues to develop.
