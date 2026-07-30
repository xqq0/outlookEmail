# Fork Upstream Sync And Image Release Spec

This spec defines the required process for updating this fork from the upstream project and publishing our own Docker image.

Use this spec whenever the task involves:

- Pulling or merging updates from `assast/outlookEmail`.
- Updating this fork after the upstream project releases a new version.
- Building or pushing `ghcr.io/xqq0/outlookemail`.
- Rebuilding the NAS deployment image from the updated fork.
- Any release workflow that could overwrite fork-only behavior.

## Repository Roles

- Upstream source: `assast/outlookEmail`.
- Our fork: `xqq0/outlookEmail`.
- Main local path: `/Users/qxq/qxq/outlookEmail`.
- Release branch for our fork: `main`.
- Upstream remote should normally be `origin`.
- Fork remote should normally be `fork`.
- Docker image should be built from our fork, not directly from upstream.
- Published image: `ghcr.io/xqq0/outlookemail`.

If remotes differ, stop and inspect before pushing:

```bash
git remote -v
```

Do not push to `assast/outlookEmail`.

## Non-Negotiable Fork Behavior

After every upstream sync, these behaviors must still work:

- Unauthenticated `GET /` returns the public landing page, not a redirect to `/login`.
- Authenticated `GET /` returns the admin UI.
- The public landing page uses `集成邮件查看器`.
- Login page uses `集成邮件查看器`.
- Admin UI uses `集成邮件管理`.
- Share page uses `集成邮件查看器`.
- Share page has no GitHub button and no GitHub outbound link.
- Share page and admin UI use `templates/partials/mail-brand-logo.html`.
- Login entry path is configurable in settings.
- Changing login entry path requires the current password.
- When a custom login path is active, old `/login` does not expose the login page.
- OAuth Client ID is configurable in settings.
- OAuth flows read Client ID through the fork's settings helper.

Never accept an upstream merge result that breaks any item above.

## Protected Files

These files have fork-specific behavior. Do not replace them wholesale with upstream versions.

High-risk files:

```text
outlook_web/segments/01_bootstrap.py
outlook_web/segments/03_mail_helpers.py
outlook_web/segments/04_routes_groups_accounts.py
outlook_web/segments/07_routes_oauth_settings_external.py
outlook_web/segments/11_routes_graph_oauth.py
static/js/index/07-settings.js
templates/partials/index/dialogs-management.html
static/js/email-share.js
templates/email_share.html
tests/test_project_runtime.py
```

UI and contract files:

```text
docs/troubleshooting.md
static/css/index/06-modals-toast.css
static/css/email-share.css
static/css/landing.css
templates/landing.html
templates/login.html
templates/index.html
templates/partials/index/layout.html
templates/partials/mail-brand-logo.html
tests/test_email_share_links.py
tests/test_system_skin_management.py
```

## Preflight

Start from the local repository:

```bash
cd /Users/qxq/qxq/outlookEmail
git status -sb
git remote -v
```

Rules:

- If the worktree has user or Claude Code UI changes, do not overwrite them.
- If there are uncommitted changes unrelated to the upstream sync, either commit/stash them with user approval or pause and ask.
- Do not commit untracked local config files such as `env.example` unless explicitly requested.
- Create a backup branch before merging upstream.

Backup branch:

```bash
git switch main
git fetch origin main
git fetch fork main
git branch "backup/custom-main-before-upstream-$(date +%Y%m%d)"
```

## Sync Upstream

Fetch and inspect upstream first:

```bash
git fetch origin main
git log --oneline --decorate --graph --max-count=12 main origin/main
git diff --name-status main..origin/main
```

Merge upstream into our `main`:

```bash
git merge --no-ff origin/main -m "merge upstream and preserve custom features"
```

Conflict rules:

- Understand upstream's new logic before resolving conflicts.
- Prefer integrating upstream improvements into our customized structure.
- Do not use a whole-file `ours` or `theirs` choice for protected files unless you have manually verified every protected behavior.
- Do not restore upstream branding.
- Do not restore share-page GitHub links.
- Do not change unauthenticated `/` back to a login redirect.
- Do not remove configurable login entry path.
- Do not remove configurable OAuth Client ID.

## Required Post-Merge Checks

Run these searches after conflict resolution:

```bash
rg -n "get_oauth_client_id|settingsOauthClientId" outlook_web static templates
rg -n "get_login_entry_path|settingsLoginEntryPath|login_entry_path" outlook_web static templates
rg -n "集成邮件管理|集成邮件查看器|mail-brand-logo" templates
rg -n "github-link|insertGithubLink|github.com/assast/outlookEmail" \
  templates/email_share.html static/js/email-share.js static/css/email-share.css
```

Expected result:

- The first three searches must find the fork customizations.
- The last search must not find GitHub references in share-page files.

Check fork diff against upstream:

```bash
git diff --stat origin/main..HEAD
git diff --name-status origin/main..HEAD
git log --oneline origin/main..HEAD
```

The diff should still include our fork-only behavior.

## Required Tests

Run focused tests:

```bash
./venv/bin/python -m unittest \
  tests.test_project_runtime \
  tests.test_email_share_links \
  tests.test_system_skin_management
```

If upstream touched mail refresh, OAuth, Graph, IMAP, database, templates, static assets, or routing, run the full suite:

```bash
./venv/bin/python -m unittest discover -s tests
```

Do not publish an image if required tests fail.

## Manual Verification

At minimum, verify the product contract:

- [ ] Unauthenticated `/` shows the public landing page.
- [ ] Authenticated `/` shows admin UI.
- [ ] Landing page displays `集成邮件查看器`.
- [ ] Login page displays `集成邮件查看器`.
- [ ] Admin UI displays `集成邮件管理`.
- [ ] Share page displays `集成邮件查看器`.
- [ ] Share page has no GitHub button.
- [ ] Share page has no GitHub link.
- [ ] Login entry path can be changed only with current password.
- [ ] Custom login entry path works.
- [ ] Old `/login` is disabled when a custom login path is active.
- [ ] OAuth Client ID setting is still loaded by OAuth flows.

## Commit And Push

Commit only the intended upstream-sync and release changes. Do not include local environment files.

```bash
git status -sb
git add <intended-files>
git commit -m "merge upstream updates and preserve fork customizations"
git push fork main
```

If the upstream sync merge already created the merge commit, verify and push:

```bash
git status -sb
git diff --quiet HEAD fork/main
git push fork main
```

Do not open a PR to the upstream project unless the user explicitly asks.

## Build Our Image

Preferred path: use the repository GitHub Actions workflow from our fork.

Trigger the workflow from `xqq0/outlookEmail` on `main`:

```bash
gh workflow run docker-build-push.yml --repo xqq0/outlookEmail --ref main
```

Watch the run:

```bash
gh run list --repo xqq0/outlookEmail --workflow docker-build-push.yml --limit 5
gh run watch <run-id> --repo xqq0/outlookEmail
```

Verify the run:

```bash
gh run view <run-id> --repo xqq0/outlookEmail \
  --json status,conclusion,url,headSha,jobs
```

The `headSha` must match our pushed `main`, not upstream `assast/outlookEmail`.

Verify the image:

```bash
docker buildx imagetools inspect ghcr.io/xqq0/outlookemail:latest
```

Expected image requirements:

- Image repository is `ghcr.io/xqq0/outlookemail`.
- Tags include `latest` and normally `main`.
- Platforms include `linux/amd64` and `linux/arm64` when the workflow supports multi-arch.
- Digest changed after a real rebuild.
- Package is associated with `xqq0/outlookEmail`.

Optional package check:

```bash
gh api /user/packages/container/outlookemail --jq '{name, visibility, repository: .repository.full_name, updated_at}'
```

## If Building Locally Is Needed

Only use local Docker build as a fallback when GitHub Actions is unavailable.

Build from our checked-out fork after tests pass:

```bash
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t ghcr.io/xqq0/outlookemail:latest \
  -t ghcr.io/xqq0/outlookemail:main \
  --push .
```

Before local push, verify GHCR login and target:

```bash
docker login ghcr.io
git remote -v
git rev-parse HEAD
```

Do not build from a direct upstream checkout.

## Release Notes

After a successful sync and image build, update:

```text
/Users/qxq/qxq/qxq笔记/集成邮件管理系统/Fork定制差异与上游同步保护.md
```

Record:

- Date.
- Upstream commit.
- Fork commit.
- Merge commit, if any.
- Test command and result.
- GitHub Actions run URL.
- Image tags.
- Image digest.
- Any changed fork-only files.
- Any new upstream behavior that might affect future merges.

If UI handoff context changes, also update:

```text
/Users/qxq/qxq/qxq笔记/mail/task.md
```

## Stop Conditions

Stop and ask the user before proceeding if:

- The fork remote does not point to `xqq0/outlookEmail`.
- The push target is upstream `assast/outlookEmail`.
- Tests fail and the cause is not understood.
- A protected feature appears removed after merge.
- GitHub Actions is building from the wrong commit.
- GHCR package points to the wrong repository.
- The worktree contains unrelated user changes that must be moved, stashed, or committed.

