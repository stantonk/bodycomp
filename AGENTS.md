# AGENTS.md

- Read `docs/REQUIREMENTS.md` before designing anything. Record decisions in `docs/`, not agent memory.
- iPhone only, Xcode 27 / Swift 6.4, SwiftUI + SwiftData. Minimum iOS is in `docs/REQUIREMENTS.md`.
- Edit `project.yml`, never `.xcodeproj` (generated, not committed). Run `xcodegen` after changing it.
- SwiftData models must stay CloudKit-compatible: every property optional or defaulted, relationships optional, no `@Attribute(.unique)`.
- Food amounts are stored in grams; convert units at the edges.
- An entry's nutrition day and meal are stored separately from its logged-at timestamp. Never derive the day from the timestamp alone.
- Never commit API keys. The OpenAI key lives in the Keychain.
- Write failing tests first (Swift Testing). Base fixtures on the real food CSV, not invented data.
- Branch off `main` and open a PR; don't commit to `main` directly.
- Commit, push, and open PRs as the agent GitHub user whose token is `GITHUB_PAT_ATHENA` in `.env` (gitignored). Never print the token or write it into git config.
- That token lacks `read:org`, so `gh pr edit` fails. Edit PRs through the REST API instead: `gh api -X PATCH repos/stantonk/bodycomp/pulls/<n> -f body=...`.
