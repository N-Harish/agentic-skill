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
- Don't edit application source without saying why. Pin base image tags and install from lockfiles.

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

Find: build step? entrypoint, listening port (grep `listen(`, `PORT`, `app.run(`, `server.port`), required env vars (`.env.example`), external services (DB/cache), health path (`/health`, `/healthz`, `/actuator/health`, else `/`), and app type: **server**, **cli** (exits), or **worker** (long-running, no port). If the port is unknown, ask once; otherwise state the assumption.

## 3. `.dockerignore` first

Write it before the Dockerfile. Always exclude: `.git`, `.env`, `.env.*` (keep `!.env.example`), `*.pem`, `*.key`, `id_rsa*`, `node_modules`, `__pycache__`, `.venv`, `target`, `bin`/`obj`, `dist`/`build` (if rebuilt in image), `.idea`, `.vscode`, `*.log`, `coverage`, `Dockerfile*`, `docker-compose*`. Never ignore the lockfile.

## 4. Dockerfile: multi-stage wherever possible

Anything needed to build or test but not to run (compilers, dev deps, source, caches) stays in earlier stages. Layout:

- Compiled or bundled (Go, Rust, Java, .NET, TypeScript): `build` → `runtime`.
- Native deps (Python wheels, Ruby gems): compilers and `-dev` headers in `builder`; runtime gets only shared libs.
- Interpreted with no build: still `deps` → `runtime`.
- Tests exist: add a `test` stage (`docker build --target test .`), not shipped.
- Frontend + backend: one named stage per toolchain, copy artifacts into one runtime.
- Single stage only when nothing needs separating (e.g. plain static files); say so in the report.

Requirements for every Dockerfile:
- First line `# syntax=docker/dockerfile:1.7`; name every stage (`AS deps|build|test|runtime`).
- `runtime` starts from a fresh slim/distroless base (never `FROM build`) and copies only the artifact.
- Copy manifest + lockfile and install **before** copying source; use `RUN --mount=type=cache` for package caches.
- Non-root `USER` with numeric UID (e.g. 10001); `COPY --chown`, not recursive chown.
- Exec-form `CMD`/`ENTRYPOINT` (`["prog","arg"]`) so the app is PID 1 and gets SIGTERM.
- Bind `0.0.0.0` (not `127.0.0.1`); `EXPOSE` the real port; `HEALTHCHECK` with a tool that exists in the image (skip on distroless).
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

Python: `builder` stage creates `/opt/venv` with `pip install -r requirements.txt`; `runtime` is `python:3.12-slim`, copies `/opt/venv`, creates user 10001, runs `gunicorn -b 0.0.0.0:8000 app:app` (never the Flask/Django dev server). Java: `maven:3.9-eclipse-temurin-21` build → `eclipse-temurin:21-jre` runtime, `JAVA_TOOL_OPTIONS=-XX:MaxRAMPercentage=75`. Rust: dummy-`main.rs` build to cache deps, then real build → `distroless/cc`. .NET: `sdk` publish → `aspnet`, `USER $APP_UID`. Static site: Node build → `nginxinc/nginx-unprivileged` (port 8080).

If a Dockerfile already exists, don't rewrite it: build it, then report problems by severity (secrets baked in > runs as root > unpinned/`latest` base > single-stage shipping build tools > no `.dockerignore` > cache-unfriendly order > no healthcheck).

## 5. Build

```bash
NAME=$(basename "$PWD" | tr 'A-Z' 'a-z' | tr -c 'a-z0-9._\n-' '-')
if git rev-parse --git-dir >/dev/null 2>&1; then
  TAG="$NAME:$(git rev-parse --short=12 HEAD)$([ -n "$(git status --porcelain)" ] && echo -dirty)"
else TAG="$NAME:local-$(date -u +%Y%m%d%H%M%S)"; fi
docker build --pull --label skill=docker-build \
  --label "org.opencontainers.image.revision=$(git rev-parse HEAD 2>/dev/null || echo unknown)" \
  -t "$TAG" -t "$NAME:latest" .
```
If a `test` stage exists, run `docker build --target test .` first; a failing suite stops the workflow. Private deps: `--secret id=npmrc,src=$HOME/.npmrc`. Target another arch: `docker buildx build --platform linux/amd64 --load`.

On build failure, read the error, fix, retry (max 3 attempts, then report). Common: `COPY` file not found → `.dockerignore` excluded it; lockfile mismatch → report it, don't swap to `npm install`; missing system lib → install in build stage only; `exec format error` → wrong `--platform`; runtime `module not found` → artifact not copied to `runtime`.

## 6. Verify (required)

Pick the mode from step 2. Use throwaway env values (`.env.example`); if a real credential is required to boot, tell the user which variable and stop. If the app needs a database, write a temporary compose file **outside the repo** with a healthchecked dependency, `docker compose up -d --wait`, probe the app, then `docker compose down -v`.

**Static checks (all modes)**
```bash
docker image inspect "$TAG" -f 'size={{.Size}} user={{.Config.User}}'   # user must be non-empty, not root/0
docker history --no-trunc "$TAG" | grep -Ei '(SECRET|PASSWORD|TOKEN|API_?KEY)[A-Z_]*=' && echo "FAIL: secret in history"
C=$(docker create "$TAG"); docker export "$C" | tar -t | grep -E '(^|/)\.git/|(^|/)\.env$|id_(rsa|ed25519)$|\.pem$' && echo "FAIL: sensitive file in image"; docker rm "$C" >/dev/null
```

**Server** (set `PORT`, `HP`, expected `200`)
```bash
CID=$(docker run -d --label skill=docker-build -p 127.0.0.1::$PORT --env-file .env.example "$TAG")
HPORT=$(docker port "$CID" $PORT/tcp | head -1 | awk -F: '{print $NF}')
for i in $(seq 60); do
  [ "$(docker inspect -f '{{.State.Running}}' "$CID")" = true ] || { echo "FAIL: exited"; docker logs --tail 30 "$CID"; break; }
  code=$(curl -s -o /dev/null -w '%{http_code}' --max-time 3 "http://127.0.0.1:$HPORT$HP")
  [ "$code" = 200 ] && { echo "PASS: $HP -> $code after ${i}s"; break; }; sleep 1
done
docker inspect -f 'health={{if .State.Health}}{{.State.Health.Status}}{{else}}none{{end}}' "$CID"
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

Fix the root cause, rebuild, re-verify (max 3 cycles, then report). Also confirm the multi-stage worked: `docker image ls "$NAME"` and `docker run --rm --entrypoint sh "$TAG" -c 'which gcc go npm mvn'` should find none (skip for distroless).

## 7. Report

Lead with the verdict, keep it short:

```
Result: PASS | PASS with warnings | FAIL
Branch: docker/containerize-20261009-1030 (from main; nothing committed)
Image:  myapp:3f2a9c1d8e4b-dirty (+ latest), 142 MB; stages: deps/build/test/prod-deps/runtime
Checks: build PASS | non-root PASS | no secrets/.git PASS | GET /healthz -> 200 in 3s PASS | graceful stop 1s PASS
Files created: Dockerfile, .dockerignore
Assumptions: port 3000 from src/server.ts:12; no DB needed to boot
Run:    docker run --rm -p 3000:3000 --env-file .env myapp:latest
```
Include how to review or discard: `git status` to review, `git switch main && git branch -D docker/containerize-20261009-1030` to throw it away (only the user decides; don't run it). Mention anything the user must supply (real env vars) and that a `-dirty` tag means uncommitted changes were built.
