# bodycomp — Requirements

Status: v1 requirements, gathered 2026-10-25 from the primary stakeholder (CEO) interview.

## Product

bodycomp is an iOS app for tracking what you eat — calories and macros — against your daily calorie
expenditure, so you can gradually lose fat and recomp your body. It understands body composition
math and is built around a user-owned food database. The core bet is that tracking compliance
requires extremely low friction, so voice logging is the primary input.

## Scope

| Item | v1 | Later |
|---|---|---|
| Audience | Personal use (the stakeholder only) | Beta group / App Store |
| Devices | iPhone only | Apple Watch voice logging (v2) |
| iPad | Not supported | — |
| Minimum OS | iOS 26 (also runs on iOS 27) | watchOS 27 for the Watch app |
| Storage | On-device (SwiftData) | iCloud sync via CloudKit |
| External food data sources | None — user builds their own database | Supplement with third-party data |

## Food database

- The user owns and builds their own database of foods and nutrition facts.
- **CSV import** via file picker. The importer and its test fixtures are designed around the
  stakeholder's real food CSV (to be provided), not an invented format.
- Each food stores nutrition per its label's serving definition (e.g. "1 slice = 28 g") so users never
  fight how a given nutrition label is written.

## Quantities and units

- Food amounts are stored in grams and **displayed in grams by default** (the stakeholder uses a food
  scale).
- Must also accept **oz, cups, tbsp, and "units"** (an egg, a slice), resolved through each food's own
  serving definitions.
- Must accept fractions and natural phrasing: "half", "one and a half", "1/3 cup", "two eggs".
- Body weight is in **lb**.

## Voice logging

- Transcription: OpenAI `gpt-transcribe`, passing the names of foods in the user's database as keyword
  hints so it recognizes exactly which food the user means.
- Interpretation: a text model maps the transcript to foods in the user's database, with amounts,
  units, meal, and nutrition day.
- **Hybrid mode (default):** the user speaks once and sees a confirmation card of matched foods,
  amounts, and macros. The app asks a follow-up question only when a food is ambiguous ("chicken thigh
  or breast?") or information is missing.
- **Silent auto-log (setting):** skip the confirmation card and save immediately, with undo/edit
  afterwards.
- Voice can target other days and meals: "yesterday's breakfast was…", "for lunch I had…".

### Unknown foods

When the user names a food that is not in their database, the app **must ask the user to add it**. It
never silently estimates. Three ways to add:

1. **Nutrition label photo** — read the label, structure the facts, user confirms.
2. **Spoken** — the user reads the facts aloud.
3. **Web search** — the app searches for the nutrition facts and the user confirms them.

## Daily log

- Every entry stores the raw **logged-at timestamp**, separately from the **nutrition day** and
  **meal** it counts toward.
- Users may log a meal later the same day or even the next day (backfill).
- Entries are grouped by meal in the UI. When the user doesn't say which meal, the voice flow can ask.
- **Day rollover:** a snack after midnight counts toward the previous day. The day starts at
  **4:00 AM** by default, configurable. *(The 4:00 AM default is an engineering assumption, not yet
  confirmed by the stakeholder.)*

## Body data

- **Manual weigh-ins only** (no HealthKit in v1), entered by typing or by **voice** (e.g. "185.4,
  18 percent body fat"). Voice weigh-ins follow the same hybrid / silent auto-log setting as food
  logging.
- Each weigh-in records **weight (lb) and body fat %**. The stakeholder has body fat % at every
  weigh-in.
- Profile: age, height, sex, activity level.

## Energy expenditure (TDEE)

### Adaptive TDEE (primary)

TDEE is back-solved from calories eaten versus the smoothed weight trend over time (the
MacroFactor-style approach). It needs roughly 2–3 weeks of data before it is reliable.

### Cold-start estimate

Until the adaptive estimate has enough data, use the method from
[tdeecalculator.net](https://tdeecalculator.net/) with its Katch-McArdle formula, replicated exactly:

- Lean body mass (kg) = weight (kg) × (1 − body fat %)
- **BMR = 370 + 21.6 × LBM (kg)**
- TDEE = BMR × activity multiplier:

| Activity level | Multiplier |
|---|---|
| Sedentary (office job) | 1.2 |
| Light exercise (1–2 days/week) | 1.375 |
| Moderate exercise (3–5 days/week) | 1.55 |
| Heavy exercise (6–7 days/week) | 1.725 |
| Athlete (2x per day) | 1.9 |

Reference check, verified against the site: 40-year-old male, 5'10", 200 lb, 20% body fat, moderate
exercise → BMR 1,938 kcal, TDEE 3,003 kcal. The site's cut and bulk targets are maintenance −500 and
+500 kcal/day.

The app hands over gradually from the cold-start estimate to the adaptive estimate as data accumulates.

## Targets

- **Goal-driven:** the user picks a goal (e.g. lose 0.5% of body weight per week, protein 1 g/lb). The
  app derives calorie and macro targets from TDEE and recalculates them weekly.
- The user can **override any target**.

## OpenAI access

- The user brings their own OpenAI API key, entered once in Settings and stored in the Keychain.
- Required model access: `gpt-transcribe`, a current text model for food matching, and web search.

## Platform landscape (as of 2026-10-25)

| | Current release | Implication |
|---|---|---|
| iOS / iPadOS | 27.0.1 (27.1 on iPhone Duo only), 27.2 RC in testing | Minimum iOS 26 (the stakeholder's iPhone is on 26.7.1); build with the iOS 27 SDK |
| watchOS | 27.0.1, 27.2 in beta | watchOS 27 for v2 |
| Xcode | 27 (Swift 6.4), released 2026-09-14 | Requires macOS Tahoe 26.6+ on Apple silicon |

## Development environment

Needed before development can start. Step-by-step instructions are in [SETUP.md](SETUP.md).

1. The development Mac (M4 Pro, 48 GB RAM) must run **macOS Tahoe 26.6 or later**. Xcode 27 does not
   run on macOS 15. Done: it is on macOS 27.0.1.
2. Install **Xcode 27** and the iOS 26 and iOS 27 simulator runtimes, run
   `sudo xcode-select -s /Applications/Xcode.app`, and accept the license.
3. Sign into Xcode with an Apple ID. A **free** Apple account is in use, which means:
   - no CloudKit (hence on-device storage only),
   - apps installed on the device expire after 7 days and must be reinstalled from Xcode.

   A paid Apple Developer Program membership ($99/yr) removes both limitations.
4. An iPhone on iOS 26 or later with Developer Mode enabled.
5. An OpenAI API key with the model access listed above.
6. The stakeholder's food CSV, committed to the repo or shared by path.
7. `xcodegen` (Homebrew), so the Xcode project is defined in a short, readable config file
   (`project.yml`) and the generated `.xcodeproj` is not committed.

## Design constraints

- SwiftData models must stay CloudKit-compatible (optional or defaulted properties, no unique
  constraints) so iCloud sync can be enabled later without a data migration.
- The deployment target is iOS 26. Use iOS 27-only APIs only when a feature needs them, behind
  `if #available(iOS 27, *)` with an iOS 26 fallback.
- Core math is test-first: Katch-McArdle, adaptive TDEE, unit conversion, and day rollover.

## Delivery plan

1. **Foundation:** food database and serving definitions, unit conversion, nutrition-day rollover,
   Katch-McArdle and targets, CSV import, manual logging screens.
2. **Voice logging:** transcription with keyword hints, food matching, confirmation card and auto-log
   setting, follow-up questions, the three unknown-food flows, and voice weigh-ins.
3. **Adaptive TDEE:** estimate from the weight trend, weekly target updates, trend views.

v2: Apple Watch voice logging, iCloud sync.

## Open assumptions

- **Day rollover at 4:00 AM** by default (configurable). Not yet confirmed.
- **Label photos** are read on-device with Apple's Vision text recognition (free, works offline), and a
  text model then structures the facts for the user to confirm.

## Sources

- [iOS 27 (Wikipedia)](https://en.wikipedia.org/wiki/IOS_27),
  [iPadOS 27](https://en.wikipedia.org/wiki/IPadOS_27), [watchOS](https://en.wikipedia.org/wiki/WatchOS)
- [Apple security releases](https://support.apple.com/en-us/100100),
  [Apple Developer beta news](https://developer.apple.com/news/?id=rfb1rooi)
- [Xcode 27 Is Out: Requirements](https://blakecrosley.com/blog/xcode-27-release),
  [Michael Tsai: Xcode 27 Announced](https://mjtsai.com/blog/2026/06/09/xcode-27-announced/)
- [OpenAI: Advancing voice intelligence](https://openai.com/index/advancing-voice-intelligence-with-new-models-in-the-api/),
  [GPT-Transcribe model docs](https://developers.openai.com/api/docs/models/gpt-transcribe)
- [tdeecalculator.net](https://tdeecalculator.net/)
