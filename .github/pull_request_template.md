<!-- PR title: use Conventional Commits, e.g. "feat(db): add tenant connection manager" -->

## Summary

<!-- What does this change and why? -->

## Linked issue

Closes #

<!-- Required. Merging this PR will close the linked issue. -->

## Checklist

- [ ] Branch named `<type>/<issue-number>-<slug>`
- [ ] Conventional Commit messages
- [ ] Tenant-scoped code resolves the tenant (no control-DB access for tenant data)
- [ ] No secrets logged or sent to the LLM
- [ ] New env vars added to `.env.example` (if any)
- [ ] Migrations included + tenant impact noted (if schema changed)
- [ ] Tests added/updated
