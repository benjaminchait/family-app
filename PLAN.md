# Family Tracking App — Plan

**Date:** 2026-07-29 (updated same day: baby born 7/27 — MVP scope confirmed as feeding + diaper changes)

**Status:** Planning — no code yet. **The baby is here (born 7/27/2026), so the tracking need is live today.** Every day without the app is a day of feeds and diapers logged on paper or memory.

**Goal:** A private iOS app for two parents to track their newborn's feeding and diaper changes, syncing between both phones via iCloud, distributed through a personal Apple Developer account. No third-party services, no accounts, no analytics.

---

## Why build instead of buy

Existing apps (Huckleberry, Baby Tracker, Glow Baby) do this well but route infant health data through third-party servers, and most monetize via subscription or data. Building keeps the data inside Apple's stack, costs nothing beyond the existing developer membership, and the scope is small enough to vibe-code. The trade-off is maintenance falls on us — acceptable for a two-user app.

---

## Usage reality (design constraints)

These constraints matter more than the feature list:

- **One-handed, half-asleep logging.** Every log action must complete in one or two taps from launch. Big tap targets, no required text entry, sensible defaults (now, most-recent feed type).
- **The headline question is "how long since the last one?"** The home screen should lead with time-since-last-feed and time-since-last-diaper, not a history list.
- **Two people log concurrently.** Whoever is holding the baby logs. Both phones must show the same state quickly, and neither parent should ever wonder "did you log that feed?"
- **Offline-first.** Logging must work with no connectivity (hospital basements, airplane mode at night) and sync when back online.
- **Editing is common.** Started the timer late, tapped the wrong side, forgot to log until morning — every entry needs easy edit and backdating.

---

## MVP scope

Confirmed 2026-07-29: the MVP is **feeding and diaper changes, nothing else**. Everything below the "deferred" line stays deferred until the MVP is in daily use on both phones.

**Feeding**

- Breast: live timer with left/right side, plus manual after-the-fact entry
- Bottle: volume (oz), contents (breast milk / formula)

**Diapers**

- Wet / dirty / both, one tap each, optional note

**Everywhere**

- Home screen: time since last feed and last diaper, plus today's counts
- History: reverse-chronological list grouped by day, edit/delete/backdate any entry
- Sync between both phones (see below)

**Explicitly deferred (v1.1+):** pumping log, sleep tracking, vitamin D / medication reminders, growth measurements, CSV/PDF export for pediatrician visits, home-screen widgets, Live Activity for the feed timer, Apple Watch app. The data model should anticipate these (a generic timestamped event) but the UI should not.

---

## Data model

Small, append-mostly event log:

- **Child** — name, birth date (known: 2026-07-27). Model as a list even though there's one child; sharing and future-proofing both want it. The MVP UI assumes exactly one child — no child picker.
- **Event** — id (UUID), type (feed / diaper), subtype (breast / bottle / wet / dirty / both), start, end (nullable, for timed feeds), side (left / right, nullable), amount (nullable, oz), note (nullable), createdBy device/parent, timestamps.

Events are independent rows that are almost never edited by both people at once, so sync conflicts are rare by construction; last-writer-wins per record is fine.

---

## The hard part: two Apple IDs, one dataset

This is the only genuinely tricky piece of the app, and it drives the framework choice.

**Verified 2026-07-29: SwiftData still has no supported CKShare (iCloud sharing) API.** SwiftData syncs a private database to one iCloud account automatically, but sharing records *between two different Apple IDs* is not exposed. Community workarounds exist but are exactly the kind of fragile glue to avoid.

Options:

1. **Core Data + `NSPersistentCloudKitContainer` share support (recommended).** Apple's supported path for sharing Core Data objects between iCloud users: one parent owns the data, creates a `CKShare`, the other accepts and gets read-write access via the shared database. Apple ships a full sample project ("Sharing Core Data objects between iCloud users"), and `UICloudSharingController` handles the invite flow. Core Data is older API than SwiftData but battle-tested, and plenty of training data exists for vibe-coding it.
2. **Raw CloudKit, shared custom zone.** Skip local persistence frameworks; write `CKRecord`s directly into a custom zone and share the whole zone once. Cleanest conceptual fit for an event log, but hand-rolling local cache, offline queueing, and sync bookkeeping recreates what `NSPersistentCloudKitContainer` gives for free.
3. **CloudKit public database.** Simple but wrong: "public" means any user of the app can query it. Ruled out on privacy grounds.
4. **Own backend (Cloudflare Workers + D1 is proven infra here).** Would work and give full control, but breaks the privacy goal — infant health data would live outside Apple's encrypted stack on infrastructure we must secure and maintain. Ruled out.

**Recommendation:** option 1. If the Core Data sharing flow fights back during the build, option 2 is the fallback — the data model is small enough that raw CloudKit stays tractable.

**Sync caveats to design around:**

- The share handshake (invite → accept) is a one-time setup flow; build a clear "setup" screen for it rather than burying it.
- CloudKit propagation is usually seconds but can lag; show local writes instantly and treat sync as eventually consistent. Silent push (CloudKit subscriptions) keeps both phones fresh.
- Both parents need iCloud signed in with iCloud Drive enabled — worth a preflight check screen.

---

## Privacy posture

- Data lives only on the two phones and in iCloud (owner's private database + shared database). No third-party SDKs, no analytics, no crash reporting services, no servers of ours.
- Use CloudKit encrypted fields (`encryptedValues`) for note text and anything free-form.
- Advanced Data Protection: if both accounts have it enabled, iCloud data gets end-to-end encryption; shared-database content follows the participants' settings. Worth enabling on both accounts regardless of this app.
- No photos in scope — keeps the data footprint boring on purpose.
- App name and bundle identifier will be visible in App Store Connect / provisioning; keep them generic (no child name).

---

## Distribution (no App Store)

Apple Developer account: benjamin@chait.co (registered, individual membership).

1. **TestFlight (recommended).** Create the app record in App Store Connect, upload builds, invite Cara by email as a tester. External-tester invites require a one-time lightweight Beta App Review of the first build; after that, new builds go out immediately. Builds expire after 90 days, so plan on re-uploading roughly quarterly — acceptable, and vibe-coding sessions will produce updates anyway.
2. **Direct install from Xcode (fallback / day one).** With a paid membership, development builds signed onto both phones last one year. Zero process, but updates mean plugging in a phone. Fine for the very first build while TestFlight is being set up.
3. **Ad hoc `.ipa`** — register both device UDIDs, sideload. Strictly worse than the other two for this case; skip.

Practical note: an individual (non-organization) membership means Cara is an external TestFlight tester, not a team member — that's fine, it only adds the one-time beta review.

---

## Stack

- **Language/UI:** Swift + SwiftUI
- **Persistence/sync:** Core Data + `NSPersistentCloudKitContainer` with CKShare (per above)
- **Deployment target:** set by the older of the two phones' iOS versions — confirm both, then target the current major version if both allow
- **Tooling:** Xcode on the existing Mac; project vibe-coded with Claude Code locally (simulator + device builds need a Mac, so this repo's remote sessions are for planning/review, not builds)
- **Repo:** this repo (`family-app`), plain Xcode project committed at the root

---

## Build sequence

1. **Walking skeleton (now — first Xcode session):** local-only app — data model, log buttons, home screen, history with edit and backdating. Install on Benjamin's phone via Xcode. Backdating matters from day one: the first days of feeds and diapers exist on paper/memory and should be entered once the app runs.
2. **Sync (next):** CloudKit container + share flow; install on Cara's phone; verify concurrent logging and offline behavior.
3. **TestFlight:** app record, first upload, beta review, Cara on TestFlight.
4. **Polish as-used:** whatever the 3 a.m. experience demands — this is where deferred features get promoted one at a time.

The baby was born 7/27 — the walking skeleton is the whole ballgame now. A local-only app on one phone tonight beats a synced app next week; until sync lands, one phone (whichever parent is primary logger) is the source of truth.

---

## Decisions needed before building

> Superseded 2026-07-29 by [QUESTIONS.md](QUESTIONS.md) — the full pre-build question list (28 items with recommended defaults) that must be answered before the one-shot build session. The five below are the headline subset.

1. **Name + bundle identifier.** Short and generic (no child name in the bundle id or App Store Connect record). Candidates to react to: Sprout, Tally, Nightshift. Bundle id something like `net.benjaminchait.<name>`.
2. **Sync approach sign-off** — Core Data + CloudKit sharing as recommended above?
3. **Distribution** — TestFlight target state with direct-install for day one?
4. **Pumping** — now decidable from actual feeding patterns rather than hypotheticals: if bottles of pumped milk are already part of the routine, pumping moves into v1; otherwise it stays deferred. (~~MVP cut line~~ — resolved 2026-07-29: MVP is feeding + diapers only.)
5. **iOS versions** on both phones — sets the deployment target.

---

## Adjacent projects (lifeops cross-reference)

- **DIY iOS Home Screen Widgets** (CLAUDE.md potential project): this app is the natural first widget surface (time-since-last-feed on the home/lock screen) and settles the "is the Apple Developer account available" prerequisite — it is.
- **Personal "Home" App — Shared Key Documents:** different data, same core problem (two Apple IDs sharing private data). Whatever CKShare experience this app produces transfers directly.
