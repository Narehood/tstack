---
name: mobile-build
description: >-
  Trigger this if the user says 'mobile build', or asks to ship, build, or
  release an Expo/EAS app for both platforms. Use for production EAS builds
  after package updates, PR merges, and green CI.
---

# Mobile Build

Run a gated production EAS build for an Expo app. Default platform is both
(`ios` and `android`). Sibling skills override the platform: `ios-build` for
iOS only (plus TestFlight), `android-build` for Android only (build only).

Target repos are Expo/EAS apps, including npm-workspace monorepos. Resolve
which app to build before anything else. Do not start updates, merges, or
builds until the app path is known.

This skill does **not** grant merge permission. List every open PR that must
merge, and get an explicit yes from the user before merging any of them.
Never force-merge, never use `--admin`, and never bypass branch protection.

## Platform

- `mobile-build`: `--platform all`. After builds succeed, submit the iOS build
  to TestFlight. Android stops after a successful production build.
- `ios-build`: `--platform ios`, then TestFlight submit.
- `android-build`: `--platform android`. No store submission.

## Preflight

Before step 1, run `scripts/preflight` from this skill's directory (use its
absolute path). Pass `--app <path>` when known, and `--platform` matching the
variant.

```bash
preflight [--app path] [--platform ios|android|all]
```

It exits 0 when ready, 1 when something is missing, 2 on bad input. Fix every
failure before continuing:

- `gh` installed and authenticated (`gh auth status`).
- `eas-cli` installed and logged in (`eas whoami`). Prefer a local
  `npx eas-cli` / project dependency when the repo pins one.
- Node and npm available; respect the repo's package manager and lockfile.
- The chosen app has `eas.json` with a `production` profile.
- Working tree is clean enough to update packages (or use a disposable
  worktree). Dirty user-owned changes are not yours to discard.

On Windows, use these scripts as-is. They are Python 3 stdlib only and do not
import `fcntl` or other Unix-only modules. Prefer `py -3` when `python3` is
missing from PATH.

## Resolve the app

If the user named an app (`datasim`, `pocket-puzzles`, `offlinearcade`,
`apps/mobile`, a path, or a slug), use that. Otherwise run
`scripts/find-apps` from the repo root:

```bash
find-apps [root]
```

It prints one app per line as `path<TAB>eas|no-eas`. When more than one app
matches, ask which to build with the harness question tool. Do not guess
across apps.

Record the app directory, its `name` / Expo slug, and that `eas.json` has
`build.production`. All later commands run with that directory as the app
cwd unless a monorepo root command is required (workspace installs).

## Steps

Each step is gated on the previous. Stop on failure; do not skip ahead to a
build.

1. **Update packages.** From the monorepo root (or the app, if it is
   standalone), bring dependencies in line with the Expo SDK and the lockfile:

   - Run `npx expo install --check` in the app. If it reports drift, run
     `npx expo install --fix` (or apply the listed fixes) so Expo-related
     packages match the SDK.
   - Update remaining dependencies with the repo's package manager in a way
     that rewrites the lockfile (`npm update`, `npm outdated` then targeted
     bumps, or the workspace equivalent). Prefer Expo-compatible versions;
     do not jump major lines the app does not already use without asking.
   - Verify integrity of what changed: review the lockfile diff, confirm
     resolved versions look sane, and run a lockfile-aware install
     (`npm ci` when a lockfile commit is clean; otherwise the install the
     repo documents). Flag unexpected publish/registry changes, preinstall
     scripts you do not trust, or packages that disappeared/reappeared under
     a new integrity hash. Stop and ask if supply-chain sanity is unclear.
   - If anything changed, commit on a branch, prove the app still resolves
     (`npx expo-doctor` when available, plus the repo's usual typecheck/lint
     for that app), and file a PR with the `file-pr` skill. Babysit it with
     `babysit-pr` until required checks are green. Do not merge it yet.

2. **Merge open PRs (with confirmation).** List every open PR against the
   default branch that should land before this release, including the update
   PR from step 1. Use `gh pr list --base <default> --state open`. For each
   PR, show number, title, author, head, and CI status.

   Ask the user, with the harness question tool (`AskUserQuestion` /
   `request_user_input`), which PRs to merge. Put the full list and your
   recommendation in the question. Merge only the PRs they explicitly
   approve. Merge with the method the repo uses. Never `--admin`, never
   `--force`, never bypass branch protection. If the repo uses a merge
   queue, stay until each approved PR reports merged.

   If the user declines all merges and an update PR is still open, stop. Do
   not build on top of unmerged release prerequisites.

3. **Confirm CI on the default branch.** After the approved merges land,
   fetch the default branch and wait for its tip to go green:

   ```bash
   ci-wait [--repo owner/name] [--ref branch] [--interval 20]
   ```

   Use `scripts/ci-wait` from this skill's directory. It prints one line per
   real event, exits 0 when checks on that ref are green, 1 when a required
   (or any, when none are required) check fails or the ref moves into a
   failing state, and 2 on bad input.

   If CI fails, stop. Report the failing checks and links. Do not start an
   EAS build.

4. **EAS production build.** From the app directory:

   ```bash
   eas build --platform <ios|android|all> --profile production
   ```

   Use non-interactive flags the logged-in CLI supports when a human cannot
   answer prompts (`--non-interactive` when available and already configured).
   Wait for the build(s) to finish. On failure, stop and report the EAS build
   URL and error. Do not submit a failed or partial build.

5. **Submit (iOS only).** When the platform includes iOS and the iOS build
   succeeded:

   ```bash
   eas submit -p ios --latest
   ```

   Prefer `--id <build-id>` when you have the build id from step 4. Android
   never submits from this skill. Stop after a successful Android build.

6. **Report.** See "Report".

## Stop conditions

Stop and report, without building or submitting, when:

- Preflight fails and you cannot fix auth/tools yourself.
- The app or `production` profile cannot be resolved.
- Package updates look supply-chain unsafe, or integrity verification fails.
- The user does not confirm the merges you need.
- A merge is blocked by branch protection or review rules you cannot satisfy
  without bypassing them.
- Default-branch CI fails or does not go green.
- An EAS build fails, is canceled, or never starts.
- iOS submit fails (report the build as done, submit as failed).

Ask the user with the harness question tool only when you cannot continue
alone: ambiguous app choice, merge confirmation, supply-chain doubt, or a
blocker outside this grant. A question in a normal chat message goes unseen
while the thread looks done.

## Permissions

`mobile-build` / `ios-build` / `android-build` authorize:

- Dependency updates and lockfile commits on a branch you create.
- Filing and babysitting PRs for those updates.
- Merging **only** after explicit per-batch (or per-PR) user confirmation.
- Running production EAS builds for the chosen platform.
- Submitting the iOS build to TestFlight when the platform includes iOS.

They do not authorize: force-merge, admin bypass, Android store submission,
changing EAS credentials/secrets, or deleting data.

## Report

End with a single report:

- **App:** path and Expo slug
- **Platform:** ios / android / all
- **Versions updated:** packages bumped (or "none")
- **PRs merged:** number, title, URL for each (or "none" / "user declined")
- **CI:** default-branch run link and verdict
- **Build URLs:** EAS build links per platform, with status
- **TestFlight:** submitted / skipped / failed (iOS only; Android is N/A)
- **Stopped early:** reason, if you did not finish
