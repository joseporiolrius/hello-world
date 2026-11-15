# S4 — containerize hello-world (APP-07)

The Session 4 lab needs **no starter import** for hello-world. You write three files yourself,
from the Session 4 slides and study guide:

- `.dockerignore` and `Dockerfile` on the branch **`s04-docker`** (lab part 1);
- `compose.yml` on the branch **`s04-compose`** (homework 2).

Each branch reaches `main` through a reviewed pull request, as in Session 3. This tag holds the
reference text of `.dockerignore` and these notes. It holds no `Dockerfile` and no `compose.yml`:
those are your work.

## The build contract

- **Base:** `ghcr.io/astral-sh/uv:python3.12-bookworm-slim` (uv and Python pinned).
- **Cache layer that does not depend on the source:** copy `pyproject.toml` and `uv.lock` first,
  run `uv sync --locked --no-install-project --no-dev`, and only then copy the source and run
  `uv sync --locked --no-dev`. A change of the source reuses the dependency layer.
- **Non-root:** the image runs as `appuser`.
- **No install at start:** `CMD ["uv", "run", "--no-sync", "hello-world"]`.
- **`.dockerignore`** keeps `.git`, `.venv`, the caches, `evidence/`, `tests/`, `docs/` and
  `fixtures/` out of the build context and out of the image.

## Run and check

```bash
time docker build -t hello-world:s4 .
docker run --rm -d --name hello -p 127.0.0.1:8000:8000 hello-world:s4
curl http://127.0.0.1:8000/          # the same JSON as the manual uv run of Session 2
docker logs hello                    # empty: with no terminal, Python keeps its output in a buffer
docker exec hello whoami             # appuser
docker stop hello
```

To see the start line in `docker logs`, run the container with `-e PYTHONUNBUFFERED=1`.
`EXPOSE 8000` is documentation only: without `-p`, `curl` answers "connection refused".

## Cache evidence (measure, do not copy a number)

After an edit of `config/settings.yaml`, a rebuild prints `CACHED` for the steps `WORKDIR`,
`COPY pyproject.toml uv.lock ./` and the first `uv sync`. The deliberate ordering mistake, one
`COPY . .` before one `uv sync`, builds without an error, but each later edit installs the
dependencies again: no `CACHED` after `COPY . .`. Record your three timings in `JOURNAL.md`.
After the drill, run `git restore config/settings.yaml` and build once more, so that the image
agrees with your source.

## Homework

- **Two containers, one image:** run `hello1` with `-p 127.0.0.1:8000:8000` and `hello2` with
  `-p 127.0.0.1:8001:8000`, and `curl` both ports.
- **Compose file** (`compose.yml`, next to the Dockerfile):

  ```yaml
  services:
    hello-world:
      build: .
      ports:
        - "127.0.0.1:8000:8000"
  ```

  `docker compose up -d`, `curl http://127.0.0.1:8000/`, `docker compose down`.

## Reviewed pull requests

The required check `CI / quality` of Session 3 (ruff, pytest, commit messages) must stay green,
and a reviewer approves before the merge. The check does not build the image: image builds join
the pipeline in Session 5.
