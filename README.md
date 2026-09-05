## CaspaHub — public releases

This repo exists for one reason: **the real app sources are private, and GitHub Actions
minutes on this account are currently blocked** on private repos ("recent account
payments have failed or your spending limit needs to be increased" — see
`github.com/settings/billing`). GitHub Actions is free/unlimited on *public* repos
regardless of that block, so builds run here instead, pulling each app's source from its
private repo via a scoped, read-only deploy token (`SOURCE_REPO_PAT`, which has read
access to both source repos below).

Two apps share this one release repo:

| App | Source repo | Tag prefix | Asset prefix |
|---|---|---|---|
| **CaspaHub** (consumer) | `ken-muritu/caspahub-flutter` (private) | `vX.Y.Z` / `auto-<sha>` | `app-<abi>-release.apk` |
| **CaspaHub Business** (operator/branch) | `ken-muritu/caspahub-business-flutter` (private) | `business-vX.Y.Z` / `business-auto-<sha>` | `caspahub-business-<abi>-release.apk` |

The tag and asset-name prefixes are what keep the two apps' in-app updaters
(`lib/services/update_checker.dart` in each app) from ever matching the other app's
release — each app's copy of that file only looks at tags/assets carrying its own
prefix. **Keep that scoping in mind before renaming anything here** — it's load-bearing,
not cosmetic.

- **Source code**: `caspahub-flutter` and `caspahub-business-flutter` (both private) —
  that's where all real development happens for each app respectively.
- **This repo**: CI config + published `.apk` releases only, nothing else. No app source
  is duplicated here beyond what a checkout step pulls in transiently during a build.
- **Once the billing block on the main account clears**, this repo becomes unnecessary —
  builds/releases can move back to each app's own repo directly. Nothing here needs
  cleaning up urgently if that happens; it can just stop being used.

## How builds happen

Four workflows, two per app (forked rather than parameterized — tags, asset names, and
the source repo all differ per app, and keeping each app's publish job textually separate
makes the update-checker scoping easy to verify by reading one file, no cross-app
conditionals to trace through):

- **`release.yml`** / **`release-business.yml`** — `workflow_dispatch` only. Checks out
  the app's source repo at a given `ref`, builds split-per-ABI release APKs, and
  publishes them as a GitHub Release here under a given `release_tag`. Run manually for
  anything you want a clean semantic version for, e.g.:
  ```
  gh workflow run release.yml --repo ken-muritu/caspahub-releases -f ref=main -f release_tag=v0.3.0
  gh workflow run release-business.yml --repo ken-muritu/caspahub-releases -f ref=main -f release_tag=business-v0.1.0
  ```
- **`poll.yml`** / **`poll-business.yml`** — run hourly (offset 30 minutes apart so they
  never race each other), plus on demand. Because each source repo's own push-triggered
  CI can't run at all right now (same billing block), a push there doesn't auto-build
  anything on its own. These workflows instead check the relevant source repo's latest
  commit on `main` against their own state file (`.state/last-built-sha.txt` /
  `.state/last-built-sha-business.txt`) in this repo; if it's new, they record the new SHA
  and fire the matching `release*.yml`, tagged `auto-<short-sha>` / `business-auto-<sha>`.
  So a regular push to either app's `main` does eventually get built and released here —
  within an hour, not immediately, and under an `auto-` tag rather than a semantic version.

Both `v*`/`auto-*` (consumer) and `business-v*`/`business-auto-*` (Business) tags coexist
in [Releases](../../releases) — cut a real semantic-version release yourself for anything
worth a proper version number; the `auto-*`/`business-auto-*` tags exist so nothing
silently goes unbuilt between those. `auto-*` and `business-auto-*` releases are always
marked prerelease, so `releases/latest` (and each app's direct-download install link)
only ever resolves to a deliberately-cut semantic release for that app.

**This does not apply to `caspahub` or `caspahub-booking`** (the Next.js web apps) — those
deploy separately via Vercel and have no relationship to this repo or to either Flutter
app's build/release cycle at all.
