# S8 — hello-world on Kubernetes (APP-13)

You import **only this `k8s/` folder** into your hello-world repository and you use the **images
that you released in Session 5**. The manifests do not hold a solution: you put your own account,
your own release tag and your own pull Secret. The cluster is the k3s of your machine after the
Session 8 replacement; `kubectl` works there without `sudo`.

## The import

A new clone knows `origin` only. Add the course repository again, then restore this folder:

```bash
git remote add upstream https://github.com/joseporiolrius/hello-world.git
git fetch upstream tag starter-s08-v1.0.1
git restore --source starter-s08-v1.0.1 -- k8s
git add k8s
git commit -m "feat: add Kubernetes manifests"
```

## The image line (step 11)

`k8s/10-deployment.yaml` has a placeholder:

```yaml
image: ghcr.io/<your-github-username>/hello-world:<your-older-release-tag>
```

Put your GitHub user name in lower case and the **older** of your two Session 5 release tags
(for example `v0.3.0`). Keep the newer tag for the rolling update. Change no other line.

## Apply order: the namespace first (steps 10 and 12)

The namespace must exist before its pull Secret and before the workloads. The token is read from
the keyboard with the echo off: it is not in a command, not on the screen and not in the shell
history.

```bash
kubectl apply -f k8s/00-namespace.yaml
read -rsp 'GHCR read:packages token: ' GHCR_PAT
kubectl -n hello create secret docker-registry ghcr-pull --docker-server=ghcr.io --docker-username=<your-github-username> --docker-password="$GHCR_PAT"
unset GHCR_PAT
kubectl apply -f k8s/
kubectl -n hello get pods -w
```

Press Ctrl-C when both pods are `1/1 Running`. The second apply of the namespace is safe: it
answers `unchanged`.

## The drill (steps 13 to 17)

1. **Two replicas answer.** Ten requests give two different pod names or more.
   Screenshot: `evidence/s8-curl-loop.png`.

   ```bash
   for i in $(seq 1 10); do curl -s http://localhost:30080/; echo; done
   ```

2. **Scale 2 to 4, then back with the file.** The file still says `replicas: 2`.

   ```bash
   kubectl -n hello scale deployment/hello-world --replicas=4
   kubectl apply -f k8s/10-deployment.yaml
   ```

3. **Self-healing.** In a second terminal run `kubectl -n hello get pods -w`. Delete one pod: a new
   one comes. Screenshot of the watch: `evidence/s8-self-healing.png`.

   ```bash
   kubectl -n hello delete pod <one-pod-name>
   ```

4. **Rolling update** to your newer tag.

   ```bash
   kubectl -n hello set image deployment/hello-world app=ghcr.io/<your-github-username>/hello-world:<your-newer-release-tag>
   kubectl -n hello rollout status deployment/hello-world
   ```

5. **A tag that does not exist, then the undo.** The new pod shows `ErrImagePull` (the pull
   failed) and then `ImagePullBackOff` (the wait before the next try). The old pods stay ready and
   serve. Screenshot of the history: `evidence/s8-rollout-history.png`.

   ```bash
   kubectl -n hello set image deployment/hello-world app=ghcr.io/<your-github-username>/hello-world:v9.9.9
   kubectl -n hello get pods
   kubectl -n hello rollout undo deployment/hello-world
   kubectl -n hello rollout history deployment/hello-world
   ```

## Repair Git: the file names the image that runs

`set image` and `rollout undo` change the cluster, not the file. Put the tag that runs into
`k8s/10-deployment.yaml`, apply it and check the pods. Then commit.

The `.gitignore` of this repository ignores `evidence/*.png`. A plain `git add evidence` adds
nothing, and `git status` does not list the pictures. Add the three screenshots by name with
`-f`, and read the staged list before the commit.

```bash
kubectl apply -f k8s/
kubectl -n hello get pods
git add k8s/10-deployment.yaml
git add -f evidence/s8-*.png
git diff --cached --stat
git commit -m "feat: pin the running release and add s08 evidence"
git push origin main
```

## Stop, never destroy

Only `k8s/` comes from this tag. Your application, your earlier evidence and the Session 5 release
pipeline stay as they are. After the lab, **stop** the machine: the cluster is on its disk, and a
destroy deletes it.
