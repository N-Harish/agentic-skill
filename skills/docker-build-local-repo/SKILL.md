---
name: docker-build-local-repo
description: Build a production-ready, multi-stage Docker image from a local git repo in any language (Node, Python, Go, Java, Rust, .NET, Ruby, PHP, static) and test that the container actually works. Use when asked to dockerize, containerize, write or fix a Dockerfile, build an image, or check whether an app runs in Docker. Detects the stack, writes .dockerignore and Dockerfile, builds with a git-based tag, then smoke-tests the running container (starts, health endpoint, graceful stop, non-root, no secrets).
license: MIT
compatibility: opencode
metadata:
  requires: docker, git, curl
---

# Docker build and verify

A successful build proves little. The job is done only when the container has been **started and probed**.

## Rules

- Never work on the user's current branch: all Dockerfile changes, builds and tests happen on a new branch (step 1).
- Never overwrite an existing `Dockerfile` or `.dockerignore`: review it and propose changes.
- Never put secrets in an image (`ENV`, `ARG`, `COPY .env`). Use runtime env vars or `RUN --mount=type=secret`.
- No `--privileged`, no docker.sock mounts, no `docker system prune`, no `docker push` unless asked. Only remove containers you created (label `skill=docker-build`).
- Don't edit application source or dependency files (`requirements.txt`, `package.json`, `pyproject.toml`) to make the image work if a Dockerfile change can do it. If an edit is unavoidable, list it prominently in the report. Pin base image tags and install from lockfiles.

## 1. Preflight and safe branch

```bash
docker version --format '{{.Server.Version}}'   # daemon reachable? if not: stop and tell the user
git rev-parse --show-toplevel && git status --porcelain
git branch --show-current                        # remember this as ORIG_BRANCH for the report
```

**Before creating or changing any file, switch to a new branch.** Use the name the user gave; otherwise `docker/containerize-$(date +%Y%m%d-%H%M)`.

```bash
git switch -c "$NEW_BRANCH"      # plain `git switch <name>` fails for a branch that doesn't exist yet; -c creates it
git branch --show-current        # confirm you are on $NEW_BRANCH before continuing
```

- If the name already exists, don't reuse it: pick another suffix.
- Uncommitted changes in the working tree come along to the new branch and are not lost. Mention this if the tree is dirty (the tag will end in `-dirty`; it will also be `-dirty` once the new Dockerfile exists uncommitted, which is expected).
- Detached HEAD: the same command works and branches from the current commit.
- Do not commit, push, merge, or switch back unless the user asks. Leave the branch checked out so they can review with `git status` (new files are untracked, so `git diff` alone won't show them).
- Repo with **no commits yet** (`git rev-parse --verify -q HEAD` fails): a new branch cannot exist without a commit (`git branch` will not list it, and `git branch <x>` errors with "not a valid object name"). **Stop and ask the user** whether to create a baseline commit of the existing files on the current branch (show `git status` first and exclude junk like `.venv`, `.opencode/`). Only after they agree: commit, then `git switch -c "$NEW_BRANCH"`. If they decline, skip the branch, list every file you create so they can delete them, use a timestamp tag, and say there is no safety net. To list branches use `git branch` or `git branch -a`, never `git branch ls`.
- Not a git repo: skip the branch, warn that there is no safety net, use a timestamp tag, and say so in the report.

Build context = repo root (monorepo: use `-f path/Dockerfile` from the root).

## 2. Analyze (answer from the code, cite the file)

| Manifest | Stack | Frozen install |
|---|---|---|
| `package.json` + lockfile | Node | `npm ci` / `pnpm i --frozen-lockfile` / `yarn --frozen-lockfile` |
| `requirements.txt`, `pyproject.toml`, `uv.lock` | Python | `pip install -r` / `uv sync --frozen` |
| `go.mod` | Go | `go mod download` |
| `pom.xml`, `build.gradle*` | Java | `mvn -B package` / `gradle build` |
| `Cargo.toml` | Rust | `cargo build --release` |
| `*.csproj` | .NET | `dotnet restore` + `publish` |
| `Gemfile`, `composer.json` | Ruby, PHP | `bundle install` (deployment), `composer install --no-dev` |

**Discovery budget: be cheap. Your first discovery action MUST be the single bash command below. Do not call Glob, Grep, or Read before it has run.** You need six facts: stack and version, how the app starts, listening port, required env vars, external services, health path (and app type: **server**, **cli**, or **worker**). Get them with one command, not by opening files one by one:

```bash
git ls-files | head -60                                   # layout; skips .venv/node_modules automatically
cat .python-version .nvmrc .tool-versions 2>/dev/null     # runtime version to match
cat pyproject.toml requirements.txt package.json go.mod Cargo.toml pom.xml 2>/dev/null | head -80   # manifests only
cat run.sh Procfile Makefile 2>/dev/null | head -40       # how it is started
grep -rnEI --exclude-dir={.git,node_modules,.venv,venv,target,dist,build,tests,test} \
  --exclude='*.lock' --exclude='package-lock.json' \
  -e 'FastAPI\(|Flask\(|express\(|createServer|SpringApplication' \
  -e 'listen\(|app\.run\(|uvicorn\.run|server\.port|PORT|EXPOSE' \
  -e 'environ|getenv|process\.env|os\.Getenv|DATABASE_URL' \
  -e '/health|/healthz|/ready|/ping|actuator' . | head -40
```

Rules:
- Read at most 3 more files, and only if the output above leaves a fact unknown: the entrypoint file and the config/settings file.
- **Never read** lockfiles (`uv.lock`, `package-lock.json`, `Cargo.lock`...), vendored or generated code, tests, or every endpoint/route file. Never `Glob "**/*"`. Don't re-read a file already seen.
- Stop as soon as the six facts are known. For anything still unknown, write down an assumption (port 8000 for FastAPI, 3000 for Node, 8080 for Go/Java) and move on: step 6 runs the container, so a wrong guess is caught there for free, which is cheaper than reading more code.
- If the port is truly unknowable and matters, ask the user once.

## 3. `.dockerignore` first

Write it before the Dockerfile. It keeps the build context small (faster builds) and keeps secrets, `.git`, and tool folders out of any `COPY`.

**Rule: ignore every dot file and dot folder, then re-include only what the build needs.** This covers `.git`, `.env*`, `.venv`, `.idea`, `.vscode`, `.github`, `.opencode`, `.claude`, `.pytest_cache`, `.next`, `.cache`, and anything similar you did not think of. Use this base (order matters: the last matching line wins, so `!` re-includes go after `**/.*`):

```
# all dot files and folders, at any depth
**/.*

# re-include ONLY dot files the build actually reads (delete lines you don't need)
!.python-version
!.nvmrc
!.mvn
!.babelrc
!.browserslistrc
!.swcrc
!.yarnrc.yml
!.yarn/releases
!.config/dotnet-tools.json

# non-dot junk: prefix with **/ so it matches at ANY depth
**/node_modules
**/__pycache__
**/*.pyc
**/target
**/bin
**/obj
**/dist
**/build
**/coverage
**/*.log
**/*.key
**/*.p12
**/id_rsa*
Dockerfile*
docker-compose*
```

`.dockerignore` patterns are matched from the context root and `*` does not cross `/`: plain `__pycache__` or `*.pyc` only match at the top level, so `app/__pycache__/` would still be copied. Use `**/` for anything that can appear in subfolders.

- Keep a `!` line only if the build uses that file: `.python-version`/`.nvmrc` (version pin read by uv/nvm), `.mvn` (Maven wrapper needs it), `.babelrc`/`.browserslistrc`/`.swcrc` (JS build config), `.yarnrc.yml` and `.yarn/releases` (Yarn Berry), `.config/dotnet-tools.json` (.NET tools). Check the build scripts and manifests; when unsure, leave it ignored and let a "file not found" build error tell you.
- Never re-include `.env`, `.env.*`, `.npmrc`, `.git`, or any key/credential dot file. Private registry auth goes through `--mount=type=secret`.
- Don't ignore `dist`/`build`/`target` if the Dockerfile copies a prebuilt artifact from them, and never ignore the lockfile. `tests/` stays in the context when a `test` stage exists.
- `Dockerfile` and `.dockerignore` are always sent to the daemon even though they are ignored, so ignoring them is safe.
- Add stack-specific ignores as needed (`.venv` and `.pytest_cache` are already covered by `**/.*`; add `*.egg-info`, `vendor/bundle`, `log`, `tmp`).

## 4. Dockerfile: multi-stage wherever possible

Anything needed to build or test but not to run (compilers, dev deps, source, caches) stays in earlier stages. Layout:

- Compiled or bundled (Go, Rust, Java, .NET, TypeScript): `build` → `runtime`.
- Native deps (Python wheels, Ruby gems): compilers and `-dev` headers in `builder`; runtime gets only shared libs.
- Interpreted with no build: still `deps` → `runtime`.
- Tests exist (`tests/`, `pytest.ini`, a `test` script, `*_test.go`...): add a `test` stage (`docker build --target test .`), not shipped. Don't put `tests/` in `.dockerignore` in that case.
- Frontend + backend: one named stage per toolchain, copy artifacts into one runtime.
- Single stage only when nothing needs separating (e.g. plain static files); say so in the report.

Requirements for every Dockerfile:
- First line `# syntax=docker/dockerfile:1.7`; name every stage (`AS deps|build|test|runtime`).
- `runtime` starts from a fresh slim/distroless base (never `FROM build`) and copies only the artifact.
- Copy manifest + lockfile and install **before** copying source; use `RUN --mount=type=cache` for package caches.
- Non-root `USER` with numeric UID (e.g. 10001); `COPY --chown`, not recursive chown.
- Exec-form `CMD`/`ENTRYPOINT` (`["prog","arg"]`) so the app is PID 1 and gets SIGTERM.
- Bind `0.0.0.0` (not `127.0.0.1`); `EXPOSE` the real port; `HEALTHCHECK` with a tool that exists in the image (skip on distroless).
- `runtime` must not ship dev dependencies, tests, docs, or tooling: `builder` installs production deps only and copies explicit paths (`COPY app ./app`), never `COPY . .`; a separate `test` stage branches from `builder`, runs the dev install, then `COPY . .` and runs the tests. Dot folders such as `.opencode` are already excluded by the `**/.*` rule in step 3.
- Never `chown -R` in a `RUN` after copying (it duplicates the whole tree in a new layer, easily +100 MB). Use `COPY --chown=10001:10001`, and create the user before the copy.
- Pin tool images too (`ghcr.io/astral-sh/uv:0.5`, not a floating tag).
- No secrets in `ENV`; `apt-get install --no-install-recommends` and remove `/var/lib/apt/lists` in the same `RUN`.

Compiled example (Go):
```dockerfile
# syntax=docker/dockerfile:1.7
FROM golang:1.23 AS build
WORKDIR /src
COPY go.mod go.sum ./
RUN --mount=type=cache,target=/go/pkg/mod go mod download
COPY . .
RUN CGO_ENABLED=0 go build -trimpath -ldflags="-s -w" -o /out/app ./cmd/app

FROM build AS test
RUN go test ./...

FROM gcr.io/distroless/static-debian12:nonroot AS runtime
COPY --from=build /out/app /app
USER nonroot:nonroot
EXPOSE 8080
ENTRYPOINT ["/app"]
```

Build-step + prod-deps example (Node/TypeScript):
```dockerfile
# syntax=docker/dockerfile:1.7
FROM node:22-slim AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN --mount=type=cache,target=/root/.npm npm ci
FROM deps AS build
COPY . .
RUN npm run build
FROM build AS test
RUN npm test
FROM deps AS prod-deps
RUN npm prune --omit=dev
FROM node:22-slim AS runtime
ENV NODE_ENV=production
WORKDIR /app
COPY --from=prod-deps --chown=node:node /app/node_modules ./node_modules
COPY --from=build --chown=node:node /app/dist ./dist
COPY --chown=node:node package.json ./
USER node
EXPOSE 3000
HEALTHCHECK --interval=30s --timeout=3s --start-period=15s \
  CMD node -e "fetch('http://127.0.0.1:3000/healthz').then(r=>process.exit(r.ok?0:1)).catch(()=>process.exit(1))"
CMD ["node","dist/server.js"]
```

Python: `builder` stage creates `/opt/venv` with `pip install -r requirements.txt`; `runtime` is `python:3.12-slim`, copies `/opt/venv`, creates user 10001, runs `gunicorn -b 0.0.0.0:8000 app:app` for WSGI apps (Flask/Django; never their dev servers). **FastAPI/Starlette are ASGI**: run `uvicorn app.main:app --host 0.0.0.0 --port 8000` (already a dependency), or gunicorn with `-k uvicorn.workers.UvicornWorker`; plain gunicorn sync workers return 500s on ASGI apps. If `uv.lock`/`poetry.lock` exists, install from it (`uv sync --frozen --no-dev`, keep `uv.lock` out of `.dockerignore`) instead of an unpinned `requirements.txt`, and take test dependencies from the lock's dev group, not ad hoc `pip install pytest`. **Writable data**: if the app writes files (SQLite, uploads, caches), create a dedicated directory owned by the runtime uid (`RUN mkdir /data && chown 10001:10001 /data`, point the config/env there, `VOLUME /data`) instead of `chown -R /app`. Tell the user data is lost unless `/data` is mounted. **Python version**: use the base image that matches `.python-version` / `requires-python` (e.g. `python:3.10-slim`); never pick a different minor version, or tests and production run on different interpreters. **uv recipe**: copy uv from `ghcr.io/astral-sh/uv`, set `UV_LINK_MODE=copy UV_PYTHON_DOWNLOADS=never`, `uv sync --frozen --no-dev` in `builder`, and in `runtime` copy the whole `/app` (venv included) from `builder` onto the **same path** with the **same base image**. Venv shebangs and the `python` symlink are absolute paths, so moving the venv (e.g. to `/opt/venv`), changing the base image, or letting uv download its own Python breaks it. Don't "fix" that with `sed`/`chmod` or by re-running `uv sync` in `runtime`; fix the cause. Java: `maven:3.9-eclipse-temurin-21` build → `eclipse-temurin:21-jre` runtime, `JAVA_TOOL_OPTIONS=-XX:MaxRAMPercentage=75`. Rust: dummy-`main.rs` build to cache deps, then real build → `distroless/cc`. .NET: `sdk` publish → `aspnet`, `USER $APP_UID`. Static site: Node build → `nginxinc/nginx-unprivileged` (port 8080).

If a Dockerfile already exists, don't rewrite it: build it, then report problems by severity (secrets baked in > runs as root > unpinned/`latest` base > single-stage shipping build tools > no `.dockerignore` > cache-unfriendly order > no healthcheck).

## 5. Build

Run this snippet verbatim. Don't hand-write the image name or tag: improvised builds produced hyphenated names, and a `-dirty` tag in the report that did not exist.

```bash
NAME=$(basename "$PWD" | tr 'A-Z' 'a-z' | tr -c 'a-z0-9._\n-' '-')
if git rev-parse --verify -q HEAD >/dev/null 2>&1; then   # needs at least one commit
  TAG="$NAME:$(git rev-parse --short=12 HEAD)$([ -n "$(git status --porcelain)" ] && echo -dirty)"
else TAG="$NAME:local-$(date -u +%Y%m%d%H%M%S)"; fi
docker build --pull --label skill=docker-build \
  --label "org.opencontainers.image.revision=$(git rev-parse HEAD 2>/dev/null || echo unknown)" \
  -t "$TAG" -t "$NAME:latest" .
```
If the repo has tests, there must be a `test` stage; run `docker build --target test .` first (do not tag it, so no stray 400 MB `:test` image is left behind) and show the result; a failing suite stops the workflow. Private deps: `--secret id=npmrc,src=$HOME/.npmrc`. Target another arch: `docker buildx build --platform linux/amd64 --load`.

On build failure, read the error, fix, retry (max 3 attempts, then report). Common: `COPY` file not found → `.dockerignore` excluded it; lockfile mismatch → report it, don't swap to `npm install`; missing system lib → install in build stage only; `exec format error` → wrong `--platform`; runtime `module not found` → artifact not copied to `runtime`.

## 6. Verify (required)

Run the blocks below **verbatim, each as one bash call**, with only `$TAG`, `$PORT`, `$HP` filled in. Do not retype or simplify them: improvised versions have measured the wrong thing (a `sleep` inside the stop timer was reported as a 5 s shutdown). If the health route is static (returns a constant and touches no database), also GET one read-only route that uses the database (e.g. a list endpoint) and require a 2xx; a passing health route alone does not prove the app works. Pick the mode from step 2. Use throwaway env values (`.env.example`); if a real credential is required to boot, tell the user which variable and stop. If the app needs a database, write a temporary compose file **outside the repo** with a healthchecked dependency, `docker compose up -d --wait`, probe the app, then `docker compose down -v`.

**Static checks (all modes)**
```bash
docker image inspect "$TAG" -f 'size={{.Size}} user={{.Config.User}}'   # user must be non-empty, not root/0
docker history --no-trunc "$TAG" | grep -Ei '(SECRET|PASSWORD|TOKEN|API_?KEY)[A-Z_]*=' && echo "FAIL: secret in history"
C=$(docker create "$TAG"); docker export "$C" | tar -t > /tmp/fs.txt; docker rm "$C" >/dev/null
# Ignore public CA bundles shipped with the OS/Python (etc/ssl, certifi, ca-certificates): they are not secrets.
HITS=$(grep -vE '^(etc/ssl/|usr/lib/ssl/|usr/share/ca-certificates/)|/certifi/' /tmp/fs.txt \
  | grep -E '(^|/)\.git/|(^|/)\.env$|(^|/)id_(rsa|ecdsa|ed25519)$|\.(key|p12|pfx)$|(priv|private)[^/]*\.pem$' | head -10)
if [ -n "$HITS" ]; then echo "FAIL: sensitive file in image:"; echo "$HITS"; else echo "PASS: no sensitive files"; fi
```

If any static check prints FAIL, look at the listed paths: decide whether each is a real secret or a false positive, and say which in the report. Never end with PASS while an unexplained FAIL is on screen.

**Server** (set `PORT`, `HP`, expected `200`)
```bash
CID=$(docker run -d --label skill=docker-build -p 127.0.0.1::$PORT --env-file .env.example "$TAG")
HPORT=$(docker port "$CID" $PORT/tcp | head -1 | awk -F: '{print $NF}')
OK=0
for i in $(seq 60); do
  [ "$(docker inspect -f '{{.State.Running}}' "$CID")" = true ] || { echo "FAIL: exited"; docker logs --tail 30 "$CID"; OK=2; break; }
  code=$(curl -s -o /dev/null -w '%{http_code}' --max-time 3 "http://127.0.0.1:$HPORT$HP")
  [ "$code" = 200 ] && { echo "PASS: $HP -> $code after ${i}s"; OK=1; break; }; sleep 1
done
[ "$OK" = 0 ] && { echo "FAIL: $HP still returned $code after 60s"; docker logs --tail 40 "$CID"; }   # a silent loop end is a failure
for i in $(seq 40); do   # wait out the HEALTHCHECK start period; "starting" is not a result
  hs=$(docker inspect -f '{{if .State.Health}}{{.State.Health.Status}}{{else}}none{{end}}' "$CID"); [ "$hs" = starting ] || break; sleep 1
done; echo "health=$hs"   # healthy = PASS, none = WARN (no HEALTHCHECK), unhealthy/starting = FAIL
S=$SECONDS; docker stop -t 10 "$CID" >/dev/null
echo "stop took $((SECONDS-S))s exit=$(docker inspect -f '{{.State.ExitCode}}' "$CID")"   # exit 137 or ~10s = SIGTERM ignored
docker rm -f "$CID" >/dev/null
```

**CLI**: `docker run --rm "$TAG" --version` (or `--help`) must exit 0 with sensible output.
**Worker**: `docker run -d ...`, `sleep 15`, confirm still running and `docker logs` shows the expected start marker, then stop as above.

| Result | Likely cause |
|---|---|
| Exits within 1-2s | missing env var, config path, file not copied to `runtime` (read `docker logs`) |
| Exit 126/127 | entrypoint not executable or wrong libc/arch |
| Exit 137 + `OOMKilled=true` | memory limit; JVM needs `MaxRAMPercentage` |
| Running but connection refused | app bound to `127.0.0.1`, or wrong port |
| Stop takes 10s / exit 137 | shell-form `CMD`, or app ignores SIGTERM |
| `permission denied` in logs | non-root user can't write; fix directory ownership |

Fix the root cause, rebuild, re-verify. Max 3 cycles: if the same kind of error survives two patches, stop patching symptoms (`sed`, `chmod`, reinstalling in `runtime`), re-read the error, and rethink the stage layout; after 3 cycles report to the user. Also confirm the multi-stage worked: `docker image ls "$NAME"` and `docker run --rm --entrypoint sh "$TAG" -c 'which gcc go npm mvn'` should find none (skip for distroless).

## 7. Report

Lead with the verdict, keep it short:

```
Result: PASS | PASS with warnings | FAIL
Branch: docker/containerize-20261009-1030 (from main; nothing committed; or: repo had no commits, so no original branch)
Image:  myapp:3f2a9c1d8e4b-dirty (+ latest), 142 MB; stages: deps/build/test/prod-deps/runtime
Checks: tests PASS (10 passed) or SKIPPED (reason) | build PASS | non-root PASS | no secrets/.git PASS | GET /healthz -> 200 in 3s PASS | graceful stop 1s PASS
Files created: Dockerfile, .dockerignore
Assumptions: port 3000 from src/server.ts:12; no DB needed to boot
Run:    docker run --rm -p 3000:3000 --env-file .env myapp:latest
```
Finish by offering (don't do it unasked) to commit the new files on the branch: `git add Dockerfile .dockerignore <any modified files> && git commit -m "Add Dockerfile"`. Explain why: uncommitted files are not tied to any branch and follow you when you switch branches, and `git branch -D` does not delete them. To review use `git status`. To discard: if committed, `git switch <original> && git branch -D <branch>`; if not committed, also `rm Dockerfile .dockerignore` and `git restore <modified files>`. Only the user decides; don't run these. List old image tags from earlier attempts (`docker image ls <name>`) so the user can `docker rmi` them. The `Run:` line must be a command you actually executed successfully (don't cite `--env-file .env.example` unless that file exists), and every number in the report must come from a command output, not an estimate. Mention anything the user must supply (real env vars) and that a `-dirty` tag means uncommitted changes were built.
