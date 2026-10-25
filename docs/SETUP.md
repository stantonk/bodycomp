# bodycomp — Development Setup

Steps to prepare a Mac and iPhone for bodycomp development. Requirements and rationale live in
[REQUIREMENTS.md](REQUIREMENTS.md).

Requires an Apple silicon Mac. Xcode 27 does not support Intel.

## 1. Back up the Mac

The macOS upgrade is a major version jump, so run a Time Machine backup first.

## 2. Upgrade macOS

Go to **System Settings → General → Software Update** and install **macOS Tahoe 26.6 or later**.
Xcode 27 does not run on older versions. The newest macOS (27) also works.

## 3. Install Xcode 27

1. Install it from the **Mac App Store**, or from developer.apple.com/download if the App Store is
   slow. It's a large download.
2. Open it once. When it asks which platforms to install, include **iOS** so the iOS simulator gets
   downloaded. Then add a runtime for the app's minimum iOS (see Scope in REQUIREMENTS.md) under
   **Xcode → Settings → Components**.
3. Point the command-line tools at Xcode and accept the license:

   ```
   sudo xcode-select -s /Applications/Xcode.app/Contents/Developer
   sudo xcodebuild -license accept
   ```

## 4. Install XcodeGen

The Xcode project is generated from `project.yml`; the `.xcodeproj` is not committed.

```
brew install xcodegen
```

Run `xcodegen` from the repo root after cloning and whenever `project.yml` changes.

## 5. Add your Apple ID to Xcode

Go to **Xcode → Settings → Accounts**, click **+**, and sign in with your Apple ID. That creates your
free personal team, which is needed to install the app on your phone.

With a free account, apps installed on the phone expire after 7 days and must be reinstalled from
Xcode, and iCloud sync (CloudKit) is unavailable. A paid Apple Developer Program membership ($99/yr)
removes both limits.

## 6. Set up your iPhone

Not needed for the simulator. Do this before working on voice logging, which is best tested on a real
phone.

1. Make sure the phone is on the app's minimum iOS or later.
2. Plug it into the Mac and tap **Trust**.
3. Open the project in Xcode once so the phone sees it. Then on the phone go to **Settings → Privacy &
   Security → Developer Mode**, turn it on, and restart.

## 7. Get an OpenAI API key

Needed for voice logging.

1. At platform.openai.com, create a project and turn on billing.
2. Generate an API key with access to `gpt-transcribe`, a current text model, and web search.
3. Enter the key in the app's Settings screen. Never commit it or paste it into chat or issue
   trackers.

## 8. Add your food CSV

Copy your food CSV into the repo; the importer and its test fixtures are built around its columns. If
it contains data you don't want in git history, keep it out of commits.

## Verify

```
xcodebuild -version         # Xcode 27.x
xcrun simctl list runtimes  # lists the minimum and current iOS
xcodegen --version
```
