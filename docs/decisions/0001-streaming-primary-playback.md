# ADR 0001: Stream from Spotify as the primary game-day playback path

**Status:** Proposed (pending resolution of open questions below)
**Date:** 2026-07-09
**Context sources:** `docs/research/spotify-streaming-viability.md`, `docs/spec-review.md`

## Context

The v0.2 spec (`docs/functional-spec.md`) makes **offline** game-day playback a hard requirement (C-5, PLAY-5, PACK-2), on the assumption that youth fields have poor/no connectivity. Research (`docs/research/spotify-streaming-viability.md`) confirmed that *if* that assumption holds, no sanctioned Spotify path can serve game day, and the product must pre-render local audio files (with gray-area ripping as the likely bytes source).

The product owner has since stated that **connectivity at the fields in question is reliably good.** That invalidates the premise behind the offline hard requirement and reopens streaming as a viable primary path.

## Decision

Adopt **streaming from Spotify via the Connect Web API as the primary game-day playback mechanism**, with a lightweight offline fallback retained (not removed).

Concretely:
- **Playback:** The operator's device runs the Spotify app logged into a single Premium account (the manager's). Plate Hype drives it via Connect REST (`PUT /me/player/play` with `position_ms`, `/seek`, `/pause`, `/volume`). This avoids the App Remote iOS screen-flip and the Web Playback SDK's iOS Safari problems.
- **Search:** Use an app-level **client-credentials** token for catalog search. This requires no user login and is **not** subject to the 5-user Development-Mode cap.
- **Authorization footprint:** Only the operator authorizes with Spotify (1 user), staying comfortably inside the 5-user Development-Mode cap. Parents do **not** authenticate with Spotify.
- **Offline safety net (retained):** Pre-cache local clips for the **active lineup only** as a fallback for momentary drops. Offline is demoted from "the requirement everything is architected around" to "a safety net," preserving the "fail before game day, never during" principle at low cost.

## Consequences

### Positive (what this simplifies)
- Removes the gray-area rip + download/transcode pipeline for the primary path (contains the C-4 risk).
- Resolves offset-source parity (`GAP-2`) for playback: the same Spotify rendition is scrubbed and played.
- Loudness normalization (PACK-3 / Q-3) is largely handled by Spotify's built-in playback normalization.
- Explicit-content flag (MOD-2) and search come free from Spotify metadata.
- Relaxes the platform decision (`DEC-1`): no ripper to ship → App Store review is not a blocker → a PWA becomes viable.

### Negative / new constraints
- **Hard dependency on Spotify Development Mode**, which is actively shrinking (5-user cap, Premium-required, preview URLs removed, 3 restriction waves in ~15 months). Treat Spotify as a disposable, isolatable dependency.
- **Premium required** on the operator account.
- **Fade-out (SONG-4 / PLAY-4) is harder** over Connect REST (volume-ramp via repeated calls is coarse, laggy, rate-limited) than with local files.
- Streaming still can't guarantee the <1s start (Q-2) on a bad day — hence the retained offline safety net.

## Open questions (must resolve before this ADR is Accepted)

1. **Parent-side preview under the cap (critical path).** Parents must hear the clip to set the drop (SONG-2/SONG-5, MUST), but rendering Spotify audio needs preview URLs (removed) or the parent's own Premium session (counts against the 5-user cap → breaks for normal team sizes). Candidate sources for parent preview: iTunes Search API 30s previews (free, no auth), Deezer 30s previews (free), or YouTube. Each reopens a *preview-only* rendition-parity question. **Needs a dedicated research pass.**
2. **Fade-out feasibility** over Connect REST — verify whether an acceptable fade is achievable, or whether fade is a local-file-only feature.
3. **Offline fallback scope** — exactly what is cached, when, and how the console chooses between streaming and cached playback without operator friction.

## Supersedes / amends

Amends the spec's treatment of offline (C-5, PLAY-5, PACK-2, Q-1) from hard requirement to primary-online-with-offline-fallback. The spec should be revised to a v0.3 reflecting this once the open questions are resolved.
