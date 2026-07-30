# Trellis Workflow

This repository is a customized fork of `assast/outlookEmail`. The fork keeps product-specific behavior and branding that must survive upstream updates.

Before changing code, read the relevant spec files under `.trellis/spec/`.

## Required Specs

Read this spec before pulling upstream updates, merging upstream releases, building Docker images, pushing GHCR images, or touching release automation:

- `.trellis/spec/fork-upstream-image-release.md`

## Default Development Flow

1. Check the worktree with `git status -sb`.
2. Treat existing uncommitted changes as user-owned unless you made them in the current task.
3. Read the relevant Trellis spec before editing.
4. Keep changes scoped to the user's request.
5. Run focused tests for touched behavior.
6. Record any new fork-only behavior in the maintenance notes.

## Protected Custom Behavior

The fork has custom behavior that must not be overwritten by upstream merges:

- Public `/` landing page for unauthenticated users.
- Configurable login entry path.
- Configurable Microsoft OAuth Client ID.
- Admin brand text: `集成邮件管理`.
- Landing brand text: `非凡的多邮箱管理系统`.
- Login and share brand text: `集成邮件查看器`.
- Share page must not contain GitHub buttons or GitHub links.
- Admin and share pages use the shared logo partial.

Canonical long-form record:

`/Users/qxq/qxq/qxq笔记/集成邮件管理系统/Fork定制差异与上游同步保护.md`
