---
name: ios-build
description: >-
  Trigger this if the user says 'ios build', or asks to ship, build, or
  release an Expo/EAS app for iOS / TestFlight only.
---

# iOS Build

Do the same gated release workflow as `mobile-build`, but only for iOS, and
submit the successful production build to TestFlight.

Read [mobile-build](../mobile-build/SKILL.md) and follow its preflight, app
resolution, package update, merge-confirmation, default-branch CI, build,
report, and stop rules. An explicit "ios build" request authorizes that work
without a separate "mobile build" request. The overrides below take
precedence.

## Platform override

Use `--platform ios` everywhere `mobile-build` says `all`:

- Preflight: `preflight --platform ios [--app path]`
- EAS build: `eas build --platform ios --profile production`
- After a successful iOS build, submit to TestFlight with
  `eas submit -p ios --latest` (or `--id <build-id>`).

Do not build or submit Android.

## Report

Same final report as `mobile-build`, with **Platform:** `ios` and a real
TestFlight status (submitted / failed). Android is N/A.
