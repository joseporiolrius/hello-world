# S6 — recover the app after a VM rebuild (recovery checkpoint)

The Session 6 lab needs **no starter import** for hello-world. The application tree does not change
at S6. This tag holds these notes only.

At S6 you destroy the workspace VM and build it again with Terraform. Your application survives
because it was **committed and released**, not because it was on the VM disk. The S5 pipeline brings
it back: `VM_HOST` and `VM_SSH_KEY` do not change, because the Elastic IP is retained and the
workspace profile puts the deploy public key on every new VM.

## What a new VM does not have

The new VM has Docker, Compose, `git`, `uv` and your two public keys. It has no clone, no GitHub
access, no registry login and no Git settings. Restore them in this order (Lab B, step 11):

1. Outbound GitHub access: a new SSH key on the VM, registered in your GitHub account.
2. Your Git identity:

   ```bash
   git config --global user.name "<Your Name>"
   git config --global user.email "<your GitHub no-reply address>"
   ```

3. The clone, its `upstream` remote and the commit hook of Session 3:

   ```bash
   git clone git@github.com:<user>/hello-world.git
   cd hello-world
   git remote add upstream https://github.com/joseporiolrius/hello-world.git
   git config core.hooksPath .githooks
   ```

4. The committed `compose.yml` of Session 5 is in the clone: `cat compose.yml`.
5. The registry login, with the token read from the keyboard:

   ```bash
   read -s TOKEN
   echo "$TOKEN" | docker login ghcr.io -u <user> --password-stdin
   unset TOKEN
   ```

6. The locked development tools: `uv sync --locked`.

## Release, do not deploy by hand (step 12)

```bash
uv run cz bump --increment PATCH
git push origin main --tags
```

- The code did not change since the last release, so a plain `uv run cz bump` answers
  `[NO_COMMITS_FOUND]`. `--increment PATCH` sets the increment by hand.
- The push carries one new tag. The pipeline runs `test`, `build-push` and `deploy`, and the deploy
  job starts the new version on the new VM.
- Prove it **on the VM**: `curl -s localhost:8000`. The rebuilt security group opens port 22 only, so
  the same request from your laptop gets no answer. Do not open port 8000 by hand: the next
  `terraform plan` shows that rule as a difference.
- Without step 5 the deploy job is red at `docker compose pull` (`denied` or `unauthorized`).

The measured recovery time runs from before `terraform apply` until hello-world and the workbench
both answer.
