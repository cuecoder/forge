# Development Workflow

Every unit of work follows the same loop: **issue → branch → commits → PR →
merge (which closes the issue)**. This keeps history traceable — every change
maps to an issue and a PR.

> Uses the [`gh`](https://cli.github.com) CLI. Authenticate once: `gh auth login`.

## The loop

```
1. Create an issue           describes the work
2. Create a branch           named <type>/<issue-number>-<slug>
3. Do the work + commit      Conventional Commits
4. Open a PR                 body contains "Closes #<issue>"
5. Merge the PR              GitHub auto-closes the issue + deletes the branch
```

## 1. Create the issue

```bash
gh issue create --title "Add tenant connection manager" --label task
# → prints the issue URL ending in the number, e.g. .../issues/42
```

## 2. Branch from up-to-date main

Branch name: `<type>/<issue-number>-<kebab-slug>`, where `type` ∈
`feat | fix | docs | refactor | chore | test`.

```bash
git switch main && git pull --ff-only
git switch -c feat/42-add-tenant-connection-manager
```

> One issue per logical unit of work. Small, focused PRs beat big ones.

## 3. Work and commit

[Conventional Commits](https://www.conventionalcommits.org/):

```bash
git add -p
git commit -m "feat(db): add TenantConnectionManager with LRU pool cache"
```

## 4. Open the PR (with the closing link)

The PR body **must** contain `Closes #<issue>` so merging closes the issue.

```bash
git push -u origin HEAD
gh pr create --fill --body "Closes #42"
```

## 5. Merge

```bash
gh pr merge --squash --delete-branch
```

The linked issue closes automatically; the branch is deleted.

## Conventions

- **Authorship** Never add AI tools (Claude, Copilot, etc.) as a
  commit `Co-Authored-By` or Contributed by, PR author, reviewer, or repository collaborator. No
  AI-attribution trailers in commit messages or PR bodies.
- **Never commit to `main`.** Always branch. Keep `main` releasable.
- **Closing keyword:** Forge uses `Closes #N` (GitHub also accepts `Fixes #N` /
  `Resolves #N`).
- **Branch type prefix** matches the commit type (`feat`, `fix`, …).
- One issue ↔ one branch ↔ one PR.

## Why this discipline

- Every commit traces to a PR; every PR traces to an issue.
- Reviewers get context (the issue) without reading the diff cold.
- Release notes can be generated from merged PRs + Conventional Commits.

See [CONTRIBUTING.md](../CONTRIBUTING.md) for code conventions and build phases.
