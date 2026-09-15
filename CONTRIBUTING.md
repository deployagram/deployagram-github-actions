# Contributing

## Repo layout

One composite action per top-level directory, each a self-contained `action.yml` (`runs: using: composite`, every `run:` step with an explicit `shell: bash`). No build step — edit `action.yml` in place. See [README.md](README.md) for what each action does and an example pipeline.

To add a new action: create a new directory with an `action.yml` (copy the structure of an existing one — `name`, `description`, `inputs`, `outputs` if any, `runs.steps`), add a row to the table in README.md, and follow the release process below to make it available.

## Versioning

Tagged like `actions/setup-java`: a full semver tag (`v1.0.0`, `v1.1.0`, ...) is the permanent, immutable release. A short major tag (`v1`) is a **moving pointer** to the latest `v1.x.y` — that's what consumers actually pin to (`uses: deployagram/deployagram-github-actions/setup@v1`), so they get non-breaking fixes automatically without editing their workflow.

This means `v1` gets force-pushed every time you cut a new `v1.x.y` release. That's expected — it's the whole point of a moving tag — but it also means anyone pinned to `@v1` picks up your change the moment you push it. Don't move `v1` until you're confident the commit is safe.

## Release process

1. Commit your change to `main` as normal.

2. Tag the commit with the real release version (annotated, not lightweight — it records who/when/why):
   ```bash
   git tag -a v1.2.3 -m "v1.2.3: <one-line summary>"
   ```

3. Move the major tag to point at the same commit:
   ```bash
   git tag -f v1
   ```

4. Push the new release tag, then force-push the moved major tag:
   ```bash
   git push origin v1.2.3
   git push origin v1 --force
   ```
   (Two separate pushes: the first is a normal new tag; the second moves an existing one, which needs `--force`.)

5. Verify both landed:
   ```bash
   gh api repos/deployagram/deployagram-github-actions/tags --jq '.[].name'
   ```

### Breaking changes

A breaking change (removed/renamed input, changed behavior an existing caller relies on) gets a new major: start a `v2.0.0`/`v2` pair instead of moving `v1`, so existing callers on `@v1` are unaffected until they deliberately opt in to `@v2`.

### First release of a new major

The steps above assume `v1` already exists. Creating a new major line (`v1` the first time, or `v2` later) is the same idea without the "move" step:
```bash
git tag -a v1.0.0 -m "v1.0.0: initial release"
git tag -a v1 -m "v1: latest v1.x.y release"
git push origin v1.0.0 v1
```
