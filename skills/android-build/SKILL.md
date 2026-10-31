---
name: android-build
description: >-
  Trigger this if the user says 'android build', or asks to ship, build, or
  release an Expo/EAS app for Android only.
---

# Android Build

Do the same gated release workflow as `mobile-build`, but only for Android.
Stop after a successful production EAS build. Do not submit to any store.

Read [mobile-build](../mobile-build/SKILL.md) and follow its preflight, app
resolution, package update, merge-confirmation, default-branch CI, build,
report, and stop rules. An explicit "android build" request authorizes that
work without a separate "mobile build" request. The overrides below take
precedence.

## Platform override

Use `--platform android` everywhere `mobile-build` says `all`:

- Preflight: `preflight --platform android [--app path]`
- EAS build: `eas build --platform android --profile production`

Never run `eas submit` for Android from this skill. Never build iOS.

## Report

Same final report as `mobile-build`, with **Platform:** `android`, Android
build URL filled in, and **TestFlight:** N/A.
