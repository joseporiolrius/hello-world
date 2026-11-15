# S5 — release pipeline: tag → GHCR → your VM (APP-08 to APP-12)

The Session 5 lab needs **no starter import** for hello-world. You write everything yourself, from
the Session 5 slides and study guide, in the `hello-world` clone on your VM:

- the `[tool.commitizen]` table of `pyproject.toml` (step 1);
- `.github/workflows/release.yml` with three chained jobs: `test`, `build-push`, `deploy` (steps 3 and 10);
- `compose.yml` on the registry image (step 6).

This tag holds these notes only. It holds no `release.yml` and no `compose.yml`: those are your work.
`hello.yml` and `ci.yml` stay as they are.

## Versions come from the commit history

```toml
[tool.commitizen]
name = "cz_conventional_commits"
version_provider = "uv"
tag_format = "v$version"
```

Keep the `allowed_prefixes` line of the starter. With the provider `uv`, the version lives in
`[project]` and the bump commit also updates `uv.lock`. With `version_files` or `pep621` the lock
file keeps the old version, and `uv sync --locked` fails in CI and in the release.

```bash
uv run cz bump --dry-run
uv run cz bump
git push origin main --tags
```

- With no `v*` tag yet, commitizen says `No tag matching configuration could be found` and asks
  `Is this the first tag created?`. That is not an error: answer `y`.
- `feat:` gives MINOR, only `fix:` gives PATCH, a breaking change gives MAJOR.
- Push your workflow commit with `git push origin main --tags` **before** the first bump. It publishes
  the `starter-*` tags of your clone. GitHub starts no workflow when more than three tags arrive in
  one push, so each later release push must carry one new tag only.

## The pipeline

| Job | Does |
|---|---|
| `test` | `uv sync --locked`, `uv run ruff check .`, `uv run pytest`, with the action versions of your `ci.yml` |
| `build-push` (`needs: test`) | logs in to `ghcr.io` with the workflow's own `GITHUB_TOKEN` and pushes `ghcr.io/<user>/hello-world:<tag>` and `:latest` |
| `deploy` (`needs: build-push`) | SSH to the VM with `appleboy/ssh-action`, then `docker compose pull` and `docker compose up -d` |

- The trigger is `on: push: tags: ["v*"]`.
- `permissions: packages: write` lets `GITHUB_TOKEN` push the image. Without it the push fails with
  a 403. The fix is never a personal token in the workflow.
- `<user>` is your GitHub user name in **lowercase**.
- A red `test` means no image and no deploy: the server keeps the old version.
- The deploy script starts with `set -eu` and `test -n "$HELLO_IMAGE_TAG"`. It never runs
  `docker compose up -d --build`: the pipeline builds once, the server only pulls.

## The Compose file on the VM

```yaml
services:
  hello-world:
    image: ghcr.io/<user>/hello-world:${HELLO_IMAGE_TAG:?tag required}
    ports:
      - "8000:8000"
    restart: unless-stopped
```

Without `HELLO_IMAGE_TAG`, every Compose command stops with `tag required`: there is no `latest`
fallback and no local build. Export the variable once for the terminal:

```bash
export HELLO_IMAGE_TAG=<your-v-tag>
docker compose pull
docker compose up -d
curl -s http://localhost:8000/
docker compose images
```

Commit this file (`feat: deploy hello-world from the GHCR image`): Session 6 restores it from Git.

## Credentials: each one has one place and one job

| Credential | Lives | Does |
|---|---|---|
| `GITHUB_TOKEN` | made by GitHub for each run | pushes the image |
| classic PAT, `read:packages` only | Docker login on the VM | pulls the private image |
| deploy key `~/.ssh/gha-deploy` | your laptop and the secret `VM_SSH_KEY` | lets the `deploy` job in |
| `VM_HOST` | repository secret | the Elastic IP of your VM |

Log the VM in to GHCR without the token in a command (it stays out of the shell history):

```bash
read -s TOKEN
echo "$TOKEN" | docker login ghcr.io -u <user> --password-stdin
unset TOKEN
```

The VM holds only the public half of the deploy key. The repository holds nothing:
`git log --all --oneline -- '*gha-deploy*'` returns nothing.

## Drills (homework)

- **Fix-only release:** one `fix:` commit, then `cz bump` chooses PATCH.
- **Safety drill:** a failing test released with `fix: exercise release failure` stops the pipeline
  at `test`. A plain `git revert` writes a `Revert "…"` message that commitizen does not count
  (`[NO_COMMITS_TO_BUMP]`), so revert in two steps:

  ```bash
  git revert --no-commit HEAD~1
  git commit -m "fix: revert the release failure test"
  uv run cz bump
  git push origin main --tags
  ```

- **Rollback drill:** export `HELLO_IMAGE_TAG` as an older tag that has an image, then
  `docker compose pull` and `docker compose up -d`. The tag of the red release has no image: its
  pull fails with `manifest unknown`. A version tag in the registry does not move: that makes the
  rollback possible.

## Evidence

`evidence/s05/`: `actions-run.png`, `curl-before.png`, `curl-after.png`, `workbench-pins.png`, and
from the homework `release-stopped.png` and `stopped.png`. `JOURNAL.md`: what you asked the AI, what
was wrong, how you verified, your lead time and the manual-step count. No credential in any picture.
