# Plate Hype — Spec Review: Gaps & Open Questions

**Reviewing:** `docs/functional-spec.md` v0.2
**Purpose:** Identify what must be resolved (or consciously deferred) before a thorough technical design can be written. This is not a critique of the requirements — the spec is unusually clear and disciplined about staying functional. It is a checklist of the ambiguities, missing decisions, and feasibility unknowns an engineer will hit on contact.

Findings are tagged `GAP-n` (something the spec should say but doesn't) or `DEC-n` (a decision — often the product owner's, not the engineer's — that gates the design). Each references the spec IDs it touches.

---

## 0. Top risks (read these first)

These three determine whether the project is buildable as written. Everything else is refinement.

1. **The catalog→offline-clip pipeline is the whole ballgame, and the two "available resources" don't obviously deliver it.** (`GAP-1`, `O-2`, `C-2`) Spotify Premium's API/SDK does not permit extracting or downloading raw audio, and its 30-second preview clips are being retired. YouTube Premium's offline downloads are locked inside the YouTube app and are not extractable. So neither stated resource, used within its terms, produces the offline, trimmable, normalizable audio file that `PACK-2`/`PACK-3` require. The realistic pipeline is *metadata search → resolve to a source (likely YouTube) → rip audio (gray-area, `C-4`) → trim → loudness-normalize → store*. The spec is right that "how" is engineering's call, but the design cannot proceed until it's confirmed which service provides *search/metadata*, which provides *the actual audio bytes*, and that these are allowed to differ. This also forces the platform/distribution decision below.

2. **Preview must play the exact audio that gets packed, or every drop is wrong on game day.** (`GAP-2`, `SONG-2`, `SONG-5`, `PACK-2`) If the parent scrubs and previews against one rendition (e.g., a Spotify stream or preview) but the pack is built from a *different* rendition (e.g., a YouTube rip — different master, live version, remix, or even a different intro length), the start offset the parent chose will not land on the same musical moment. The spec's preview requirement implicitly demands **offset-source parity**: the offset is only meaningful relative to a specific, identified rendition, and that rendition must be the one that ends up in the pack.

3. **Platform and distribution are unstated but heavily constrained, and they cascade into auth, notifications, offline storage, and audio control.** (`DEC-1`) "Offline playback + precise audio session control + background/interruption handling + local media storage + push notifications" is exactly the feature set where a PWA (especially on iOS) is weakest and a native app is strongest — but a native app built around gray-area audio ripping cannot pass App Store / Play Store review, which pushes toward personal/sideload/TestFlight distribution. The spec's non-commercial, small-scale framing (`C-3`) is compatible with that, but the engineer needs the target platform(s) and distribution channel decided before choosing almost anything else.

---

## 1. Identity, Auth & Sessions

- **DEC-2 (`INV-2`, `O-4`):** No-password onboarding needs an *identity anchor*. Is it email (magic link), a social/OAuth provider (Google/Apple), phone OTP, or passkeys? `INV-3` pre-assigns players by **email**, which strongly implies email is the anchor — confirm, and decide what happens for a parent who has only a phone number.
- **GAP-3 (`PLAY-5`, `DATA-4`):** The console must work **offline**, so auth/session state must be cached on-device and survive with no network. How long is a session valid offline? How is the operator's identity trusted at a field with no connectivity? This needs an explicit stance (e.g., long-lived device session established while online, re-validated opportunistically).
- **GAP-4:** **Account recovery / device change** is unaddressed. With magic-link or device-bound auth, what happens when a parent or manager gets a new phone mid-season? Without a password there must be a defined re-link path.
- **GAP-5:** The manager's own sign-in method is never stated. Same passwordless mechanism as parents, or something stronger given they hold delete/moderation power (`ROLE-3`, `TEAM-6`)?

## 2. Roster & Player Model

- **GAP-6 (`Parent Link`, `ROLE-2`):** A player can have **multiple parents**, but there is no **conflict policy**. If two linked parents edit the same child's song, who wins — last-write, or is one designated primary? This is a real scenario (separated households) and affects the data model, not just UI.
- **GAP-7 (`INV-3`):** Can a player have **zero parents** and be fully manager-managed (song set entirely via the fallback path in `MOD-3`)? The moderation flow implies yes, but the claim/onboarding flow assumes a parent exists. Confirm both paths are first-class.
- **GAP-8 (`TEAM-2`, `PLAY-1`):** Relationship between **active/bench status, roster size, and batting-order slot** is ambiguous. A 12-player roster typically bats 9–10 with subs. Is "batting order" a subset of active players? What orders the console when `TEAM-4` (batting order, a SHOULD) is absent — jersey number, alphabetical, entry order? `PLAY-1` says "in batting order" as if it always exists.
- **GAP-9 (`TEAM-3`, `PLAY-1`, `PLAY-6`):** Benched players are "excluded from the default console view but remain configurable." When a bench player **subs in mid-game**, how does the operator reach their control quickly? `PLAY-6` ("select any player out of order") may cover it, but the interaction for surfacing an excluded player should be explicit.
- **GAP-10:** Is **jersey number unique** within a team? The console labels controls by name + number (`PLAY-1`); duplicate/blank numbers need a defined behavior.
- **GAP-11:** **Position(s)** are captured but never used by any requirement. Confirm they're intentional metadata (fine) rather than an implied-but-missing feature.

## 3. Invitations & Claims (privacy-sensitive)

- **GAP-12 (`INV-1`, `INV-4`, `O-5`):** A **non-consumable shareable link/code** means anyone who obtains it can join the team and potentially **claim any unclaimed player** — i.e., a stranger (or the wrong parent) could attach themselves to a child and gain edit rights and visibility into that child's config. Given the users are minors, the spec needs a **claim-verification stance**: does a manager approve claims? Does email pre-assignment (`INV-3`) become mandatory rather than optional for the open-link case? This is the single biggest privacy hole in the current draft.
- **GAP-13 (`INV-4`):** When an invitation is **revoked or regenerated**, are already-joined parents unaffected (presumably yes)? State it, so revocation isn't mistaken for a kill-switch on existing access.

## 4. Song Assignment & Catalog

- **GAP-14 (`SONG-1`):** **Which catalog does search query**, and is it the same source as the audio? If search is Spotify metadata but audio comes from YouTube, the app must **resolve one to the other**, and mismatches (wrong version, live/remix, cover) need a resolution/confirmation step (ties back to `GAP-2`).
- **GAP-15 (`SONG-2`):** Scrubbing to a drop implies access to the **full track** at arbitrary offsets (drops are often past 60s), which a 30s preview cannot provide and which offline packing complicates. Feasibility of "scrub the whole track" against the chosen source must be confirmed.
- **GAP-16 (`MOD-2`, `Song Assignment` explicit flag):** The **explicit-content flag has no defined source**. Spotify exposes it; a YouTube rip generally does not. If audio resolves to YouTube, explicit detection is unsolved — decide whether the flag is best-effort from search metadata, manager-judgment only, or both.
- **GAP-17 (`SONG-3`, `SONG-4`, `INTRO-1`):** Missing bounds/definitions: **minimum clip length**, **fade-out duration** ("short" is undefined), and **intro maximum length** ("a few seconds"). These need concrete numbers to implement and to size packs.
- **GAP-18 (`SONG-7`, `PACK-2`):** Are **saved alternates** packed offline too, or only the active assignment? Affects pack size and prep time.

## 5. Moderation & Approval State Machine

- **GAP-19 (`SONG` states, `MOD-4`, `PACK-2`):** The **assignment lifecycle is never enumerated**. Implied states: *unset → set → (pending → approved | rejected) → eligible*. "Eligible" (what `PACK-2` packs) needs a precise definition = *active player + (approved OR approval-not-required) + successfully resolvable*. Write the state machine down.
- **GAP-20 (`MOD-4`, big loophole):** Does **editing an already-approved song reset it to pending**? If not, a parent can get an innocuous song approved and then swap in something inappropriate before game day. The re-moderation trigger must be defined.
- **GAP-21 (`SONG-6` vs `PACK-5`):** `SONG-6` says a parent can change the song "before a game pack is prepared." After prep, are parent edits **blocked**, or **allowed but inert until re-prep**? This lock/refresh window needs a clear rule, since it governs the relationship between parent edits and `PACK-5`.

## 6. Game Pack Preparation

- **GAP-22 (`PACK-1`, `PACK-2`, `O-2`):** **Where does the heavy processing run** — download, transcode, trim, normalize? On the phone (heavy, battery, storage) or a personal server/desktop (introduces a deployment component the spec never mentions)? This is the deployment envelope the technical design must define, and it interacts with `DEC-1`.
- **GAP-23 (`PACK-3`, `Q-3`):** Loudness normalization needs a **target and reference** (e.g., a fixed LUFS target such as −16 LUFS, normalized to a fixed reference rather than relative to the current pack) so consistency holds across re-preps and across packs, not just within one batch.
- **GAP-24 (`PACK-4`):** **Enumerate failure reasons** surfaced per player (not found, resolution/rip blocked, track too short for the chosen offset+length, normalization failed) so the operator/manager knows how to fix each.
- **GAP-25 (`PACK-5`, `PACK-7`, offline):** The **prepared pack is the source of truth on game day**, but if a parent/manager changes a song after prep, the offline console can't know. Confirm the console always plays the pack as-prepared and that **stale-pack warning** happens at prep/pre-game time (`PACK-7`), never as a surprise at the plate.
- **GAP-26 (`C-4`, "who pulls the trigger"):** The gray-area rip is presumably executed at **pack-prep time by a manager**, not by parents (who only search/preview). Confirm this, because it concentrates the terms-of-use risk on one account/device and keeps parents clear of it — which matches "keep the risk contained."

## 7. Game-Day Console & Audio (the actual failure surface)

- **GAP-27 (concurrency):** If the operator **taps player B while A is still playing**, what happens — immediate interrupt with fade, hard cut, or ignore until A finishes? Common in real use; unspecified.
- **GAP-28 (audio session realities):** Nothing addresses the things that actually break field playback: **Bluetooth speaker pairing/latency, output routing, the iOS silent switch, phone-call/notification interruptions, another app seizing the audio session, and system volume level.** `Q-7` ("never leaves the console stuck") implicitly requires graceful handling of interruptions — make it an explicit requirement, since these are the most likely game-day failures.
- **GAP-29 (`PLAY-9`, `LIVE-2`):** Clarify **soundcheck** behavior: does it play full clips or truncated, auto-advance or manual step, at what volume? And note that soundcheck's auto-advance is an intentional **exception** to the `LIVE-2` "human trigger per play" invariant (soundcheck is not a walk-up).
- **GAP-30 (`Q-2`):** The **<1s start latency** is a MUST but its baseline is undefined: cold app launch vs warm, first tap vs subsequent, with intro-then-song sequencing. Define the measurement conditions so it's testable (per `AC-2`).

## 8. Assisted Operation & Live State

- **GAP-31 (`LIVE-1`, `GC-3`):** Live state provides "current and on-deck"; `LIVE-1` arms "the next batter." Define the **event→arm mapping**: does the app arm the on-deck batter (so the clip is ready when they become current) and what live transition triggers advancing the highlight? Low-risk since it's arm-only, but the timing should be pinned.
- **GAP-32 (`GC-1`, `GC-4`):** **Identity matching between the GameChanger roster and the Plate Hype roster** (names, jersey numbers — often inconsistent) is unspecified. Does import *create* the roster (clean) or *merge* into an existing one (fuzzy-match problem)? Song assignments are keyed to Plate Hype players, so a bad match sends the wrong song.
- **GAP-33 (`GC-1`, `O-1`, security):** GameChanger has no public API, so integration likely requires the **manager's GameChanger credentials or session**. Storing/handling third-party credentials is a security and privacy concern that needs a stance (and interacts with `O-5`).

## 9. Notifications

- **GAP-34 (`MOD-5`, `INV-5`, `SONG-9`):** **No notification channel is specified** — email, push, SMS, or in-app only? This is gated by `DEC-1`: reliable push on an iOS PWA is historically weak, and a passwordless email-anchored design may make email the only dependable channel. Decide the channel(s) before promising `MOD-5` notifications.
- **GAP-35 (`SONG-9`):** The "**manager-visible deadline**" implies a deadline concept in the model. Who sets it, is it per-team or per-game, and is it merely a visual cue or does it drive reminders/auto-fallback?

## 10. Privacy, Minors & Data Lifecycle

- **GAP-36 (`O-5`, `TEAM-6`):** `TEAM-6` deletes "roster and assignments" — but does deletion also remove **recorded child voice intros, packed media, cached copies on operator/parent devices, and song history (`DATA-3`)**? Voice recordings of minors and children's names/jersey numbers are the sensitive assets; the deletion scope and offline-copy cleanup must be explicit. Recommend a short data-inventory + retention/deletion policy even for personal use.
- **GAP-37 (`O-5`):** Define **what PII exists and where it lives** (child name/nickname/number, parent email, voice intros), especially the offline copies that by design live on the operator's phone (`PACK-2`).

## 11. Sync, Multi-Device & Reuse

- **GAP-38:** **Source of truth and sync/conflict model** across devices (parent's phone edits, manager's moderation view, operator's packed device) is undefined. Confirm the scope of "offline" is **console playback only** — parents and managers edit while online — so the only offline artifact is the prepared pack. If parent offline editing is ever expected, that's a much bigger sync problem to flag now.
- **GAP-39 (`DATA-2`, `DATA-3`, `TEAM-5`):** **Season/roster carry-forward** semantics: does duplicating a team copy players + active assignments + alternates + history + parent links, and does it re-verify/re-pack audio, or reuse existing clips? Retention window for `DATA-3` history is also unstated.

## 12. Cross-cutting / smaller notes

- **GAP-40 (`C-4`, "fail before game day"):** The "fail in advance, not during the game" principle deserves a first-class **pre-game readiness check / checklist** requirement (beyond `PACK-7`), e.g., a single green/red gate covering pack readiness, audio-output test, and (if used) live-source connectivity — run before first pitch.
- **GAP-41:** No **observability/logging** surface for the builder to diagnose failures after the fact ("why did that clip fail to prep?"). Minor, but useful given the gray-area moving parts.
- **GAP-42 (`INTRO-4`):** A **text announcer intro** is offered as an alternative to recorded audio — but text implies **text-to-speech** at pack time (another dependency, possibly online) or on-device TTS quality/offline concerns. If offered, its production path needs the same offline guarantee as audio intros (`INTRO-3`).

---

## Decision summary

**Product-owner decisions (not the engineer's to make alone):**
- `DEC-1` — Target platform(s) and distribution channel (native iOS/Android vs PWA; store vs sideload/TestFlight). Gates auth, notifications, offline storage, audio control.
- `DEC-2` — Identity anchor / sign-in method (email magic link vs OAuth vs phone OTP vs passkey).
- `GAP-6` — Multi-parent conflict/primary policy.
- `GAP-12` — Claim verification model for open invite links (given minors).
- `GAP-20`/`GAP-21` — Re-moderation-on-edit and the parent-edit lock window.
- `GAP-36`/`GAP-37` — PII inventory, retention, and deletion (incl. offline copies and voice intros).

**Feasibility assessments the engineer must complete before committing to the design:**
- `GAP-1`/`GAP-2`/`GAP-14`/`GAP-15` — The search-metadata vs audio-bytes split, offset-source parity, and full-track scrub feasibility (the core pipeline).
- `GAP-16` — Explicit-content flag source when audio ≠ metadata source.
- `GAP-22`/`GAP-23` — Where processing runs, and the loudness target/reference.
- `GAP-32`/`GAP-33` — GameChanger roster matching and credential handling (already correctly scoped SHOULD/MAY by `O-1`).

**Recommended spec additions (functional, in-scope for a v0.3):**
- Assignment-lifecycle state machine (`GAP-19`).
- Concurrency + audio-interruption behavior on the console (`GAP-27`, `GAP-28`).
- Concrete bounds: min clip length, fade duration, intro max length (`GAP-17`).
- Console ordering fallback when no batting order exists, and reaching benched subs (`GAP-8`, `GAP-9`).
- Pre-game readiness gate (`GAP-40`).
