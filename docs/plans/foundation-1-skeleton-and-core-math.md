# Foundation PR 1 — App skeleton and core math

Status: plan, 2026-10-25. First of three PRs in the Foundation rung of
[REQUIREMENTS.md](../REQUIREMENTS.md#delivery-plan):

1. **App skeleton and core math (this PR)**
2. Food database: food model, serving definitions, unit conversion, CSV import (needs the food CSV)
3. Manual screens: logging by meal, weigh-ins, profile, goal-driven targets

## Goal

An app that builds and runs on the minimum and current iOS simulators, with the two pieces of core
math that don't depend on the food CSV: the cold-start TDEE estimate and nutrition-day rollover, both
test-first.

## Out of scope

SwiftData models, settings storage, and UI beyond a placeholder screen. Those come with the PRs that
first need them. This PR's math takes plain inputs (weight, body fat, timestamp, rollover hour) so the
later models and settings can feed it.

## Milestones

### 1. Project skeleton

- A local Swift package, `BodyCompCore`, for logic that doesn't need the phone (the math in this
  PR). Its tests (`BodyCompCoreTests`, Swift Testing) run on the Mac with `swift test`, with no
  simulator. Platforms: the minimum iOS from REQUIREMENTS.md, plus macOS so the tests can run on the
  Mac.
- `project.yml` with one app target (`BodyComp`) that depends on `BodyCompCore`. Deployment target is
  the minimum iOS, built with Xcode 27 in Swift 6 language mode. No app test target yet; it's added
  with the first app code that needs testing.
- Bundle ID `com.stantonk.bodycomp`.
- A placeholder SwiftUI screen.
- Ignore `*.xcodeproj` and the package's `.build/` in `.gitignore`.
- Add a "Run the tests" section to `SETUP.md` with the `swift test` command.
- Add a line to `AGENTS.md`: logic that doesn't need the phone goes in `BodyCompCore`, so its tests
  run on the Mac.

Done when `swift test` runs in `BodyCompCore`, and `xcodegen` then `xcodebuild build` succeeds on an
iPhone simulator.

Signing for a real phone (team ID for the free personal team) isn't needed for the simulator. It's
deferred to the first PR that needs a device.

### 2. Cold-start TDEE (Katch-McArdle)

Tests first, in `BodyCompCoreTests`, then the implementation.

- `ActivityLevel`: sedentary 1.2, light 1.375, moderate 1.55, heavy 1.725, athlete 1.9.
- BMR from weight in lb and body fat in percent (`20` means 20%):
  LBM kg = lb × 0.45359237 × (1 − bf / 100); BMR = 370 + 21.6 × LBM.
- TDEE = BMR × activity multiplier.
- Results are unrounded; rounding happens only for display.

Test cases:

- Reference case from tdeecalculator.net: 200 lb, 20% body fat, moderate → BMR rounds to 1,938 and
  TDEE rounds to 3,003. Rounding BMR before multiplying would give 3,004, so this case also guards the
  "round only at display" rule.
- Each activity level's multiplier.
- 0% body fat (BMR from full body weight) as a boundary check.

### 3. Nutrition-day rollover

Tests first, then the implementation.

- Given a timestamp, a rollover hour (default 4, configurable in whole hours 0–23) and a calendar with
  a time zone, return the nutrition day as a calendar date (year, month, day).
- Before the rollover hour, the entry counts toward the previous calendar day. At or after it, the
  same day.
- Compares the local clock hour to the rollover hour. It does not subtract 4 hours from the
  timestamp: on the spring-forward day the night is only 23 hours long, so 4:30 AM minus 4 hours
  lands at 11:30 PM the day before, which is wrong.
- How the day is persisted is decided with the log entry model in PR 3. This function only computes
  the default day; the voice and manual flows can still override it (backfill).

Test cases. Every test passes its time zone explicitly, so none depend on the simulator's zone.

- Rollover boundaries, run as one test with a list of time zones as arguments: 3:59 AM → previous
  day; 4:00 AM → same day; 11:30 PM → same day. The zones:
  - America/Chicago (UTC−6/−5, US daylight saving)
  - Europe/London (UTC+0 in winter)
  - Asia/Kolkata (UTC+5:30)
  - Asia/Kathmandu (UTC+5:45)
  - Pacific/Auckland (UTC+13 in its summer)
  - Pacific/Honolulu (UTC−10, no daylight saving)
- The same instant gives a different day in different zones. 2027-03-10 09:00 UTC is 3:00 AM in
  Chicago (→ March 9), 9:00 AM in London (→ March 10) and 10:00 PM in Auckland (→ March 10).
- Just after midnight on the 1st of a month and on January 1 → last day of the previous month or year.
- Daylight saving, US: Chicago spring-forward day (2027-03-14), 4:30 AM → March 14. Fall-back day
  (2027-11-07), 1:30 AM → November 6.
- Daylight saving, southern hemisphere: Sydney spring-forward day (2027-10-03), 4:30 AM → October 3.
- Rollover hour 0 → the nutrition day equals the calendar day.

### 4. Verify on both iOS versions and open the PR

- `swift test` passes on the Mac.
- Run the same `BodyCompCore` tests once on an iOS 26 simulator (iPhone 17, iOS 26.2) and an iOS 27
  simulator (iPhone 17, iOS 27.0), to catch any difference between the Mac and iOS.
- The app builds for both simulators.
- Open the PR with this plan linked.

## Assumptions

- **Body fat is passed as a percent** (`20`, not `0.20`), matching how it's spoken and typed.
- **The rollover is set in whole hours.** Minutes can be added later if needed.
- **The nutrition day is computed once, in the time zone where the entry is logged, and stored with
  the entry** (PR 3). Traveling later never moves an existing entry to a different day.
