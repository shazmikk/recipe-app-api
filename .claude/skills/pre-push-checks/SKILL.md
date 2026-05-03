---
name: pre-push-checks
description: Run the same tests + lint that .github/workflows/checks.yml runs in CI, before a push to main is attempted or before claiming code is "ready to push." TRIGGER when the user says they are about to push, asks to push, asks if code is ready to push/merge to main, or when you are about to tell the user that code is ready to push/merge to main. Mirrors the CI Checks workflow so local results match GitHub Actions.
---

# Pre-push checks for `main`

This project gates `main` on a GitHub Actions workflow at `.github/workflows/checks.yml` that runs two Docker-Compose commands. Before any push to `main` — and before telling the user that code is "ready to push" — run the **exact same commands locally** so the local result matches CI.

## When to invoke this skill

Invoke (or follow these steps without being asked) in any of these situations:

- The user says they are about to push, asks to push, or asks "is this ready to push/merge?"
- The user is about to open a PR targeting `main`.
- You are about to tell the user that a change is "ready," "good to go," "ready to ship," or "ready for review/merge."
- The user mentions CI is failing on `main` and wants to reproduce locally.

Do **not** invoke for unrelated work (refactor in progress, exploratory edits, partial fixes the user explicitly said are WIP).

## The checks (run in order, both must pass)

Run from the repo root. Service name is `app` (defined in [docker-compose.yml](../../../docker-compose.yml)).

### 1. Tests

```powershell
docker-compose run --rm app sh -c "python manage.py test"
```

### 2. Lint

```powershell
docker-compose run --rm app sh -c "flake8 ."
```

These match exactly what runs in CI (`.github/workflows/checks.yml`), so any failure here is a guaranteed CI failure.

## Preconditions to verify first

Before running the checks, confirm:

1. **Docker Desktop is running.** If `docker info` errors with a pipe / daemon message, ask the user to start Docker Desktop and wait for the engine to be ready.
2. **The image is built / up to date.** If the Dockerfile or `requirements.txt` / `requirements.dev.txt` changed since the last build, run:
   ```powershell
   docker-compose build
   ```
   first. Otherwise stale dependencies will produce misleading test/lint results.
3. **`flake8` is installed in the image.** It comes from `requirements.dev.txt`, which is only installed when the build arg `DEV=true` is set. The compose file already sets `DEV=true`, so a fresh `docker-compose build` is sufficient. If `flake8: not found` appears, the image was built without the dev arg — rebuild.

## How to interpret results

- **Both commands exit 0** → CI will pass. You may tell the user the code is ready to push.
- **Tests fail** → Report the failing test names and the assertion / traceback. Do not claim ready. Offer to investigate.
- **Lint fails** → Report each `flake8` violation with `path:line:col: code message`. Many are auto-fixable; offer to fix and re-run.
- **Container fails to start / command not found** → This is the same class of bug as the earlier `sh -c` issue. Check `docker-compose.yml` `command:` syntax and that `app` service is healthy. Do **not** report the code as ready until the checks actually ran to completion.

## What to say to the user

- **On success:** State both checks passed and reference what ran ("tests + flake8 in the app container, same as CI"). Then it is safe to push.
- **On failure:** State which check failed and surface the relevant output. Do **not** soften with "should be ready after a small fix" — say it is not ready and what blocks it.
- **If preconditions blocked you** (Docker not running, image not built): Say so explicitly. Do not claim the checks passed when they did not actually run.

## Why this exists

The CI workflow at `.github/workflows/checks.yml` is the merge gate for `main`. Running the same commands locally with `docker-compose run --rm app` guarantees parity with CI: same image, same Python, same dependency versions, same working directory. Running `pytest` or `flake8` directly on the host does **not** count — host Python and host packages diverge from the container and will hide real CI failures.
