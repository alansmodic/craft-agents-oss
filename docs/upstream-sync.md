# Keeping this fork in sync with upstream

This fork tracks [`craft-ai-agents/craft-agents-oss`](https://github.com/craft-ai-agents/craft-agents-oss)
and carries a small set of local changes (Railway deploy config, `Dockerfile.server`,
`README.md`, a regenerated `bun.lock`).

## How auto-sync works

`.github/workflows/sync-upstream.yml` runs every Monday at 06:17 UTC (and can be
run on demand from the **Actions** tab → **Sync upstream** → **Run workflow**). Each run:

1. Fetches upstream `main` and checks whether this fork is behind.
2. Merges upstream onto a `sync/upstream-<timestamp>` branch.
3. **Regenerates `bun.lock`** with the pinned bun version (`BUN_VERSION`, kept in
   step with `pr-check.yml`) so the `lockfile-frozen` check passes. The lockfile is
   never copied from either side — this fork's dependency set differs from upstream's.
4. Opens a PR and enables **auto-merge**. When the required checks
   (`lockfile-frozen`, `docker-build`) go green, GitHub merges it automatically.
5. If a real (non-lockfile) conflict occurs, it opens a **draft PR** labelled
   `sync-conflict` with the conflict markers in place for manual resolution, and the
   run fails so you get notified.

## Why a PAT is required (`SYNC_PAT`)

`main` is a protected branch: changes must go through a PR and pass the required
checks. The built-in `GITHUB_TOKEN` **cannot create pull requests** on this repo,
and any PR it did create would **not trigger** the required checks (a GitHub safety
rule against workflows triggering workflows). Both problems disappear when the
workflow authenticates as a real user via a Personal Access Token.

### One-time setup

1. Create a **fine-grained PAT**: GitHub → **Settings** → **Developer settings** →
   **Personal access tokens** → **Fine-grained tokens** → **Generate new token**.
   - **Resource owner:** `alansmodic`
   - **Repository access:** Only select repositories → `alansmodic/craft-agents-oss`
   - **Permissions:** Repository permissions →
     - **Contents:** Read and write
     - **Pull requests:** Read and write
   - Expiration: your preference (set a calendar reminder to rotate it).
2. Copy the token.
3. Add it as an Actions secret: repo → **Settings** → **Secrets and variables** →
   **Actions** → **New repository secret** →
   - **Name:** `SYNC_PAT`
   - **Secret:** *paste the token*
4. Trigger a run to confirm: **Actions** → **Sync upstream** → **Run workflow**.

Until `SYNC_PAT` exists, the workflow fails fast with a clear message on the first step.

## Manual sync (no automation)

You can always catch up by hand:

```bash
git remote add upstream https://github.com/craft-ai-agents/craft-agents-oss.git   # once
git fetch upstream main
git checkout -b sync/manual main
git merge upstream/main
bun install --ignore-scripts        # regenerate bun.lock with the pinned bun version
git add bun.lock && git commit --no-edit
git push origin sync/manual
gh pr create --base main --fill && gh pr merge --auto --merge
```
