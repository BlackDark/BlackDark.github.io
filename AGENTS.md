# Personal website

Static personal site and Hugo blog deployed to GitHub Pages.

## Verify

CI installs the blog with frozen pnpm, builds Hugo, and runs actionlint and zizmor on the deployment workflow. The aggregate job is `check`. Only main-branch builds deploy Pages.

## Git

- Conventional Commits, one topic per PR.
- Squash unless commits need separate review.
- Force-push only with `--force-with-lease` on own branches.

## Boundaries

- Do not deploy Pages from a local session.
- Keep pnpm and Node versions exact; do not use floating `latest` or `lts/*` in CI.
