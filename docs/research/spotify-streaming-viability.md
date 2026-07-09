# Research Memo: Can Plate Hype stream directly from Spotify instead of downloading clips?

**Date:** 2026-07-09
**Question investigated:** The spec assumes walk-up clips are pre-downloaded/rendered to local files for offline game-day playback. Is that assumption necessary, or could we instead **stream directly from Spotify at game time** — possibly by running the Spotify app in the background and controlling it from Plate Hype?
**Method:** Four parallel research passes over primary sources (Spotify developer docs, changelog/blog, developer forum, GitHub SDK issues) plus real-world walk-up-music apps, 2019–2026.

---

## Bottom line

**No. Streaming from Spotify cannot be the game-day playback path, and the "run Spotify in the background and control it" approach fails specifically on the two things Plate Hype needs most: offline reliability and clean in-app triggering.** The spec's original instinct — pre-render each clip to a local, app-owned file and play *that* at the field — is correct and is the same conclusion every incumbent walk-up app has reached. Spotify's realistic role shrinks to **discovery/search metadata only**, and even that is now sharply limited. The audio bytes must come from somewhere Plate Hype controls.

This *reinforces* the offline hard requirement (C-5, PLAY-5, PACK-2) rather than relaxing it.

---

## Why each Spotify path fails for game-day playback

### 1. App Remote SDK — "run Spotify in the background and control it" (the idea you raised)
This is the closest thing to what you described: the official Spotify app does the playback/caching/networking, and Plate Hype sends it commands (play URI, seek to ms). It works on Android, online. But for our use case it has **three disqualifiers**:

- **iOS foregrounds Spotify on every play.** Apple's SDK requires an "app switch" to wake the Spotify app when starting/resuming playback — **the screen flips away from Plate Hype to the Spotify app.** Unacceptable for a live console where the operator taps a player and expects the clip to fire in-app.
- **It is not reliably offline.** `connect()` performs an online authorization check. Spotify's documented "offline for up to 24h" allowance is contradicted by its own GitHub issues (#175, #192), where the handshake fails with *"Spotify must be online to verify this authorization request"* within **minutes** of losing signal (e.g., walking into a garage). A youth field with weak signal is exactly this scenario.
- **No usable volume control** → no programmatic fade-out (SONG-4, PLAY-4). Local volume API doesn't exist; the Connect volume API is reported broken while App Remote is connected.

Also: connection auto-drops after ~30s of no playback, requiring reconnect. Maintained but slow-moving; recurring reliability complaints 2023–2026.

### 2. Web Playback SDK — browser/PWA streaming player
- **Streams only. No offline, ever.** It's a browser-based Spotify Connect device; audio is DRM-protected and streamed live. Cannot cache/save. Requires continuous network + full Premium.
- **iOS Safari is chronically unreliable.** Autoplay policy blocks programmatic start (`activateElement()` widely reported ineffective → double-tap workarounds); resume/transfer reported broken on Safari in 2024–2025.
- **`setVolume()` does not work on iOS at all** → again, no fade-out on iPhone.

### 3. Web API (Connect endpoints) — control playback via REST
- Player endpoints (`/play`, `/seek`, `/volume`) survive and are stable, **but require Premium and an already-active Spotify device** — the API controls existing devices, it can't conjure playback from nothing. That means a live network connection and a running Spotify instance. Not offline.

### 4. Live streaming reliability at a weak-signal field
- Spotify publishes no sub-second start guarantee. On a genuinely weak/intermittent connection, a cold stream must resolve auth + metadata, open a connection, and fill a buffer before audio — routinely **multiple seconds, and can fail outright** when the link drops. This cannot meet the **<1s start** target (Q-2), which is only achievable if the audio bytes are already on the device.

---

## Collateral finding: the Spotify **Web API** is also a shaky foundation on its own

Even limiting Spotify to search/discovery, the platform has become hostile to hobby apps (three restriction waves in ~15 months):

- **Development Mode is now capped at 5 authorized users** (down from 25), the **app owner must hold Spotify Premium**, and it's 1 Client ID per developer (effective Feb 2026). Extended quota (to lift the cap) is **org-only, ~250k-MAU gated** — unreachable for a personal app. If only the *operator's* device authenticates, 5 users is livable; if every coach/parent logs into Spotify, we hit a wall with no upgrade path.
- **30-second preview URLs were removed** for new/dev-mode apps (Nov 2024). This matters directly: we **cannot use Spotify previews to let a parent scrub a clip** (SONG-2/SONG-5). Search still works but now returns ≤10 results/request.
- Spotify's own docs say Dev Mode "should not be relied on as a foundation for building." Developer sentiment 2024–2026 is openly negative.

---

## What the incumbents actually do

- **BallparkDJ** (leading baseball walk-up app): **no public Spotify integration** — Spotify won't grant DJ-app access. It plays **local files (MP3/AAC/M4A/WAV), device-library/iTunes purchases, and bundled royalty-free clips that work with no WiFi.**
- **SportSongs**: explicitly engineers around flaky venue WiFi and **pre-warms audio for reliable playback on slow networks.**
- **All major services are equally constrained.** Apple Music / MusicKit gives no third-party programmatic offline playback of downloaded catalog tracks either; downloads are DRM-locked to the first-party app. Guaranteed offline = local files you own.

---

## Implications for the spec and technical design

1. **Keep offline pre-rendering as a hard requirement (C-5, PACK-2). It is vindicated, not optional.** Game-day playback plays app-owned local audio files, full stop.
2. **Spotify's role is (at most) search/metadata for discovery** — and even that is capped (5 users, ≤10 results, no previews) and may erode further. Treat any Spotify dependency as disposable and isolatable.
3. **The audio-bytes source is still the unsolved core question (spec `GAP-1`/`O-2`).** Since neither Spotify nor Apple Music can legally supply extractable clip audio, the realistic sources are: (a) a gray-area rip from YouTube/other (the risk `C-4` already accepts), (b) user-supplied/owned local files, or (c) a licensed/royalty-free library. This needs an explicit decision.
4. **Scrubbing/preview (SONG-2/5) can't lean on Spotify previews** (removed). Preview UX must be built against whatever the actual audio source is — which reinforces spec `GAP-2` (offset-source parity: the parent must scrub the *same* rendition that gets packed).
5. **Because store apps can't ship gray-area ripping, this again points at the platform/distribution decision** (spec `DEC-1`): personal/sideload/TestFlight distribution, and the ripping step run by the manager at pack-prep time, not by parents (spec `GAP-26`).

---

## Confidence

High on all disqualifying facts (offline-connect failure, iOS app-switch, no iOS volume, streams-only SDK, dev-mode caps, preview removal, incumbent behavior) — all from primary Spotify docs, official SDK repos/issues, and Spotify's own blog. Not independently verified: exact play/seek latency numbers (Spotify publishes none) and whether a private Spotify *partnership* (unavailable to hobby projects) could change the offline picture. Full source URLs are captured in the four research agent transcripts for this session.
