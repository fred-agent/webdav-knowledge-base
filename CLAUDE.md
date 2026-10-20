# CLAUDE.md — webdav-knowledge-base

Audience: AI coding assistants and the developers driving them. Read this, then
the README's "Standalone development" and "What a run does".

## What this repository is

One Fred **Knowledge Base**: a pod that mirrors the documents of a folder
published over WebDAV into a Fred library. It is built on
`fred-sdk[knowledge-base]` and ships as an image and a Helm chart. It was
extracted from `fred-samples` (`knowledge-bases/webdav`); nothing else lives
here.

| Path | What |
|---|---|
| `webdav_kb/` | The package. `knowledge_base.py` (declaration and handler) is the authoring surface; `synchronize.py` is one run. |
| `tests/` | Offline suite (no network, `pytest-socket`). |
| `config/` | `configuration.yaml` (a shape to copy, pointed at `localhost`) and `env.template` (the one secret). |
| `local-webdav/`, `dockerfiles/Dockerfile.webdav-share` | An Apache `mod_dav` share to run against (`make share-run`). A test fixture, never deployed. |
| `Dockerfile` | The deployable image: `publish`, then `run`. |
| `deploy/values.yaml` | What every deployment starts from, for Fred's generic `knowledge-base` chart (no chart of its own). |
| `deploy/k3d/` | The contract with fred-deployment-factory's local k3d (`make k3d-app DIR=<this checkout>`). |
| `.github/workflows/` | Publishes the image to `ghcr.io/fred-agent`: a release on `code/vX.Y.Z`, `swift-dev` on every push to `swift`. |

## Commands

From the repository root; `make help` lists everything.

```bash
make dev test code-quality          # locked deps from PyPI, offline tests, ruff/bandit/detect-secrets/basedpyright
make share-run && make sync URL=http://localhost:8088/dav/   # one run against the local share, no Fred
make docker-build docker-smoke      # the image, checked offline
make dev-local                      # fred's libraries from your checkout at FRED_REPO (no default); make dev-pypi goes back
```

## What must stay true

- **The committed `uv.lock` resolves fred from PyPI.** No `[tool.uv.sources]`. To work on
  unreleased fred, `FRED_LIBS=local` with `FRED_REPO` (no default) overlays your checkout on
  the environment (`make dev-local`), the image (`docker-build`) and the k3d chart
  (fred-deployment-factory's docs/LOCAL-DEVELOPMENT.md, "Work on unreleased fred"). The floor is
  `fred-sdk[knowledge-base]>=4.5.0`; raise it only to a released version.
- **Deletion is earned.** A run retracts a document only after a complete
  listing of the share (`reconciliation: complete`). A bounded or interrupted
  walk reports `partial` and retracts nothing. This is the rule most tests
  guard; never weaken it to make a test pass.
- **The pod keeps no state.** Fred's library is the record: versions are the
  share's entity tags, stored and read back through the SDK.
- **Configuration is one YAML.** The SDK's keys (`app`, `knowledge_base`,
  `security`, `scheduler`, `observability`) plus this pod's own `webdav:`
  section. `app.runtime_id` is required. The environment carries the client
  secret only, named by `security.m2m.secret_env_var`.
- **Identifiers do not move.** Package `webdav_kb`, definition
  `fred.webdav.folder`, prefix `fred.webdav`, client `kb-fred.webdav`:
  Fred binds the prefix to the first client that publishes it, so renaming
  any of them breaks existing deployments.
- **Versions are tied to tags.** `code/vX.Y.Z` must equal
  `pyproject.toml`'s version, and so must `deploy/values.yaml`'s `image.tag`; `make
  docker-build` builds that same reference locally. Never create a tag unless asked: it
  publishes what other people deploy.
- **Secrets never enter git.** `config/.env` is ignored; a deployment reads the
  client secret from a Kubernetes Secret (`extraEnvVars` in `deploy/values.yaml`).

## Working rules

- Before calling a change done: `make test code-quality`; for the image, also
  `make docker-build docker-smoke`; for `deploy/`, `make k3d-app-validate DIR=<this
  checkout>` from fred-deployment-factory (with `FRED_LIBS=local` until fred releases the
  chart).
- Keep the README the single place for how the pod behaves; do not create
  other documents.
- Commits: `Type(scope): what changed` (`Feat`, `Fix`, `Chore`, `Docs`), one
  logical change each, no co-author line.
