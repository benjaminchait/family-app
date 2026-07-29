# One-Shot MVP — Questions to Answer First

**Date:** 2026-07-29

**Purpose:** Everything that must be decided before writing any Xcode/Swift, so a single build session can produce the MVP without stopping to ask questions. Every question has a recommended default — answer by exception ("accept all defaults except #12: …"). Once answered, the next artifact is `SPEC.md` (screen-by-screen behavior, data schema, entitlements checklist), which becomes the actual one-shot build prompt.

A question with no answer and no default marked **[blocker]** stops the build; everything else has a workable default.

---

## A. Environment facts

These are facts to collect, not decisions. Wrong assumptions here are the classic one-shot killers (API not available on the installed Xcode, deployment target above a phone's iOS version, signing failures).

**1. Mac: macOS version and installed Xcode version?** [blocker]
Determines the Swift/SwiftUI API surface the code can use.
*Default: none — run `sw_vers` and `xcodebuild -version` and paste the output.*

**2. Both iPhones: model and iOS version?** (Settings → General → About) [blocker]
Sets the deployment target — the app targets the older of the two.
*Default: none — needs the two version numbers.*

**3. Apple Developer membership: is it the paid Individual program (not just a free account), and does App Store Connect load for benjamin@chait.co?**
CloudKit entitlements and TestFlight both require the paid program. Note the Team ID (Membership page) for the spec.
*Default: assume paid and active (stated as registered); confirm Team ID before the build session.*

**4. iCloud state on both phones: signed in, iCloud Drive enabled, and enough free iCloud storage?** Also: is Advanced Data Protection enabled on either account?
Sync silently fails without iCloud Drive. Advanced Data Protection doesn't block CloudKit but is worth knowing for the privacy posture.
*Default: assume yes/yes/yes; check Advanced Data Protection status when convenient.*

**5. Does Cara's phone ever connect to the Mac (cable or same Wi-Fi) for the day-one direct install, or is she TestFlight-only?**
Determines whether both phones get the app in session one or hers waits for TestFlight.
*Default: her phone can connect at least once; both installed in session one.*

---

## B. Identity (baked into entitlements — hard to change later)

**6. App name?** [blocker]
Shows on the home screen and in App Store Connect. Short, generic, no child name.
*Default: pick from Sprout / Tally / Nightshift, or supply your own. No default — this is taste.*

**7. Bundle identifier?**
*Default: `net.benjaminchait.<lowercased app name>`.*

**8. iCloud container identifier?**
*Default: `iCloud.<bundle id>` — the standard convention; no reason to deviate.*

---

## C. One-shot scope

**9. Is CloudKit sync + the share flow inside the one-shot, or is the one-shot local-only?**
The riskiest scoping call. Retrofitting CloudKit onto an existing store is where migration pain lives, so the store must be CloudKit-backed from the first line regardless.
*Default: code includes CloudKit private-database sync AND the share (invite Cara) flow, but the one-shot's definition of done is local behavior on one phone. Sync working across both phones is verified — and debugged if needed — as its own follow-up session. CloudKit share flows are notoriously fiddly on first run; don't let that block shipping the logger.*

**10. Project layout: plain Xcode project committed to this repo?**
*Default: yes — plain `.xcodeproj` at the repo root, no XcodeGen/Tuist. Two-person app; tool ceremony isn't worth it.*

**11. Automated tests in the one-shot?**
*Default: unit tests only for the pure logic — "time since last", daily counts, day-boundary math. No UI tests. Hand-testing on the phone is the real bar.*

**12. Does the one-shot include an app icon?**
*Default: yes, a simple generated flat icon — a missing icon makes the app feel broken on the home screen. Replace later at leisure.*

---

## D. Feeding — exact behavior

**13. Breast feeding entry: live timer, manual after-the-fact entry, or both?**
*Default: both — timer as the primary flow, manual entry (start time + duration) for backdating and forgotten feeds.*

**14. What does a breast feed record capture: total duration + side(s), or per-side durations?**
Per-side timing doubles the timer UI complexity (switch button, two accumulators).
*Default: total duration plus which side(s) were used (left / right / both), single timer. Track per-side minutes only if you already know you want them.*

**15. Should the app suggest which side to start on next ("last fed: left → start right")?**
Cheap to compute, genuinely useful at 3 a.m.
*Default: yes — shown as a hint on the home screen, never enforced.*

**16. "Time since last feed": measured from the start or the end of the previous feed?**
Pediatric convention counts start-to-start ("feed every 2–3 hours" means start times). This defines the biggest number on the home screen — decide it, don't discover it.
*Default: from the start of the last feed, labeled clearly.*

**17. Running timer semantics: what happens if the app is killed mid-feed, and what about a forgotten timer?**
*Default: timer = persisted start timestamp, so it survives app kill/relaunch. If a timer has run over 90 minutes, the app flags it and asks for the real end time instead of silently logging a marathon feed.*

**18. Bottle: units and required fields?**
*Default: ounces in 0.25 oz steps; contents is a required two-button choice (breast milk / formula) defaulting to whatever was logged last. If you think in ml, say so now — mixed units are a data mess.*

**19. Pumping: in or out of v1?** Decidable from the actual routine this week.
Answer this from what's really happening: are pumped-milk bottles part of the rotation already?
*Default: out — add in v1.1 once feeding routines have settled. The bottle "breast milk" option covers consumption; pumping output tracking is the only thing deferred.*

---

## E. Diapers — exact behavior

**20. Diaper record: is wet / dirty / both (one tap each) sufficient, or do you want any detail fields (color/consistency)?**
The pediatrician's first-weeks question is counts per day — wet/dirty answers it. Detail fields slow down every single log.
*Default: three one-tap buttons, optional free-text note for anything unusual, no structured detail fields.*

---

## F. Conventions shared by everything

**21. When does "today" start for the daily counts?**
*Default: midnight, device local time. (Some parents prefer a 7 a.m. boundary so night feeds group with the preceding day — only pick this if you already know you think that way.)*

**22. Time display format?**
*Default: relative for recency ("2h 15m ago") plus absolute times in 12-hour format ("3:42 AM") in the history list.*

**23. Notes field on every event?**
*Default: yes, optional, never required, never on the fast path — reachable from the entry's detail/edit view.*

**24. Record who logged each event, and show it?**
*Default: record it always (device-based — Benjamin's phone vs Cara's phone), display it only in the detail view, not in the list.*

**25. Editing and deleting: any guardrails?**
*Default: every field editable including timestamps (backdating is a first-class flow); delete requires a confirmation tap; no undo in v1.*

**26. The paper log from the first days: entered one-by-one, or is a bulk-entry screen needed?**
*Default: one-by-one via the manual/backdated entry flow — it's only a few days of records. A bulk screen is one-shot scope creep.*

---

## G. Home screen — the contract

**27. Sign off on exactly this home screen:**

- Top: two large tiles — **time since last feed** (with side hint per #15) and **time since last diaper**, each also showing the event's clock time.
- Middle: today's counts — feeds (breast/bottle split) and diapers (wet/dirty split).
- Bottom: the fast-log controls — **Left / Right / Bottle / Wet / Dirty / Both**, one tap to start (timer) or log (instant), visible without scrolling.
- If a feed timer is running, it replaces the feed tile with elapsed time + stop control.
- History is one tab/swipe away, never in front of logging.

*Default: as written. Edits here are cheap now and expensive after the one-shot.*

---

## H. Definition of done for the one-shot

**28. Sign off on these acceptance criteria:**

1. Builds clean and installs on Benjamin's phone via Xcode.
2. Every event type loggable in at most two taps from app open (timer feeds: two taps to start, one to stop).
3. Manual entry works with arbitrary past timestamps; edit and delete work on every record.
4. Home screen numbers are correct, including across the "today" boundary and after backdated entries.
5. All data survives force-quit, reboot, and airplane mode; nothing requires network.
6. CloudKit-backed store with sync + share code present (per #9), even though cross-device verification is a follow-up session.
7. Unit tests for the stats logic pass.

*Default: as written.*

---

## What happens after the answers

1. Answers land in this file (or in chat — they'll be recorded here either way).
2. `SPEC.md` gets written: screen-by-screen behavior, Core Data schema, entitlement/capability checklist, project settings (deployment target, bundle id, container id, Team ID), and the ordered build steps.
3. The one-shot session runs on the Mac with Xcode: Claude Code writes the project against `SPEC.md`, and the session ends with the app on a phone.
