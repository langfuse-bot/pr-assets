# pr-assets

Image hosting for screenshots embedded in pull requests opened by the
`langfuse-bot` automation.

## Why this repo exists

GitHub's drag-and-drop upload in the PR composer (the thing that produces
`https://github.com/user-attachments/assets/...` URLs) is a **browser-session-only**
endpoint. It is not reachable with a personal access token or the `gh` CLI, so an
automated agent cannot use it.

This repo is the token-only substitute. Images are committed here through the
GitHub Contents API (`repo` scope is enough) and referenced from PR bodies as
`https://raw.githubusercontent.com/...` URLs.

Because this repo is **public**, those URLs are served anonymously as
`image/png`, and `github.com`'s Content-Security-Policy allows
`*.githubusercontent.com` under `img-src`. GitHub's markdown renderer therefore
emits a plain `<img>` tag that loads for **any logged-out visitor** — no camo
proxy, no auth, no redirect.

## Layout

```
pr/<pr-or-issue-number>/<label>-<UTC-timestamp>.png
```

The timestamp makes every upload collision-free, so the Contents API never has
to do a read-modify-write (updating an existing path would require passing its
blob `sha`).

## Usage

Uploads go through `skills/langfuse-implement-pr/scripts/upload-pr-asset.sh` in
the agent workspace. It uploads the file, verifies the result is publicly
readable as an image, and prints a ready-to-paste markdown snippet:

```
./upload-pr-asset.sh shot.png 1234 "trace detail dark"
# → ![trace-detail-dark](https://raw.githubusercontent.com/langfuse-bot/pr-assets/<commit-sha>/pr/1234/trace-detail-dark-20260930T132318Z.png)
```

URLs are pinned to the **commit SHA** rather than to `main`, so a published PR
body can never be invalidated by later changes to this repo.

## Constraints

- **Keep images under 10 MiB.** Above exactly 10 MiB (10,485,760 bytes)
  `raw.githubusercontent.com` switches from `Content-Type: image/png` to
  `application/octet-stream`. Chromium still renders it, but the correct content
  type is safer across other clients.
- The Contents API accepts up to ~20 MiB per file; ~30 MiB returns `502` and
  larger returns `422`. Screenshots should be far below this anyway.
- Nothing here is ever deleted on a schedule, but history can be squashed to a
  single commit if the repo grows — published PR URLs pinned to old commit SHAs
  would break, so prune only assets from long-closed PRs.

## Contents

Everything in this repo is machine-generated build/PR evidence. It is not code
and is not part of any Langfuse product.
