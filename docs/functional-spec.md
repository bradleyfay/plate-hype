# Plate Hype: Functional Specification

**Product name:** Plate Hype (working name)
**Version:** 0.2 (draft for technical design)
**Status:** Functional requirements only. Contains no architectural or implementation decisions by design. Technology, data modeling, integration mechanics, and system design are the engineer's realm and are explicitly out of scope for this document.

**Requirement language:** MUST = required for the product to be considered working. SHOULD = strongly desired, omit only with a stated reason. MAY = optional / later. Every requirement is tagged with an ID for reference during technical design.

---

## 1. Overview

A walk-up song is a short music clip played over a speaker as a player walks to the plate in baseball or softball. The clip typically starts partway into a track (the "drop") and plays for a few seconds until the player is set.

Plate Hype lets a youth team run walk-up music on game day. A team manager builds a roster and invites parents. Each parent picks and configures their own child's walk-up song. On game day, an operator plays each player's clip through a speaker, with the option to have the current batting order drive which song is queued next.

The core problem being solved is reliability. The product must do the ordinary things (managing songs, inviting parents, playing a clip when a player walks up) without breaking, and it must keep working at a field with poor or no connectivity. Streaming-service flakiness must not affect game-day playback.

---

## 2. Goals and Non-Goals

### 2.1 Goals
- Let parents self-serve the selection and configuration of their own child's walk-up song.
- Give the game-day operator a fast, dependable console for playing clips.
- Keep game-day playback fully functional with no network connectivity.
- Optionally use the live batting order to reduce the operator's manual work, without ever removing operator control.
- Reuse songs and rosters across games and across a season.

### 2.2 Non-Goals
- Not a commercial or publicly distributed product. Personal / club use only.
- Not a general music player or a replacement for a streaming app.
- No social feed, discovery, likes, or public sharing of songs.
- No monetization, ads, merchandise, or in-app purchases.
- No support for sports other than baseball / softball in this version.

---

## 3. Users and Roles

| Role | Description |
|------|-------------|
| **Team Manager** | Creates and owns a team. Full control over roster, invitations, song moderation, game-pack preparation, and the game console. |
| **Co-Manager** | Optional. Holds the same permissions as the Team Manager for a given team. A team MAY have more than one. |
| **Parent** | Joins a team by invitation. Manages the walk-up song(s) for their own child(ren) only. Cannot see or edit other children's configuration and cannot operate the game console. |

**Role rules**
- **ROLE-1 (MUST):** A single user account MAY belong to multiple teams, in any combination of roles across teams.
- **ROLE-2 (MUST):** A parent's edit and view access is scoped to the specific player(s) they are linked to. They MUST NOT be able to view or modify any other player's song, offset, intro, or status.
- **ROLE-3 (MUST):** Only a Team Manager or Co-Manager may operate the game console, moderate songs, and prepare a game pack.
- **ROLE-4 (SHOULD):** A Team Manager can promote a parent to Co-Manager and demote a Co-Manager back to parent.

---

## 4. Core Concepts

Described functionally. These are the things the product must keep track of; how they are stored or modeled is out of scope.

- **Team:** A named group with one or more managers and a roster of players. Belongs to a season or can be carried across seasons.
- **Player (Roster Entry):** A child on the team. Relevant information: name (and optional preferred display name / nickname), jersey number, position(s), batting-order slot, active/bench status, and links to the parent(s) who manage them.
- **Parent Link:** The association between a user account and one or more players, granting that user edit rights over those players' song configuration.
- **Song Assignment:** The walk-up configuration for one player. Relevant information: the selected track reference (title and artist), a start offset (where the clip begins), a clip length (how long it plays), an optional fade-out, an explicit-content flag, an approval status set by a manager, and an optional recorded intro. A player has at most one active song assignment at a time, and MAY have a small saved set of alternates.
- **Intro:** An optional short spoken audio clip that plays immediately before a player's song (for example a coach or the child announcing the name).
- **Lineup:** The batting order for a given game. May be entered manually or imported. Drives the sequence and the "on-deck" arming behavior.
- **Game Pack:** The prepared, ready-for-offline-use set of all clips (songs and intros) for a team, along with the readiness status of each. Prepared before a game so that everything plays without a network connection.
- **Live Game State:** When available from an external source, the current batter and on-deck batter during a game in progress.

---

## 5. Functional Requirements

### 5.A Team and Roster Management

- **TEAM-1 (MUST):** A Team Manager can create a team with a name and season label.
- **TEAM-2 (MUST):** A manager can add, edit, and remove players. Each player has at minimum a name; jersey number, position, and batting-order slot are optional.
- **TEAM-3 (MUST):** A manager can mark a player active or benched. Benched players are excluded from the default game console view but remain configurable.
- **TEAM-4 (SHOULD):** A manager can set and reorder the batting order manually.
- **TEAM-5 (SHOULD):** A manager can duplicate a prior team or roster to start a new season without re-entering players.
- **TEAM-6 (MUST):** A manager can delete a team, with confirmation, removing its roster and assignments.

### 5.B Invitations and Parent Onboarding

- **INV-1 (MUST):** A manager can generate an invitation to a team that can be shared through common channels (for example a link or a scannable code posted in a team chat or shown at practice).
- **INV-2 (MUST):** Onboarding MUST be low-friction and MUST NOT require the parent to create or remember a password.
- **INV-3 (MUST):** During onboarding a parent can claim the player(s) they are responsible for from the team roster. A manager MAY pre-assign a player to a specific parent email so the claim is one tap.
- **INV-4 (SHOULD):** A single invitation can onboard multiple parents (it is not consumed by the first use), and a manager can revoke or regenerate it.
- **INV-5 (SHOULD):** A manager can see, per player, whether a parent has joined and whether a song has been set, so gaps are visible at a glance.
- **INV-6 (MAY):** A parent with more than one child on the same or different teams can manage all of them from one account.

### 5.C Song Assignment (Parent-Facing)

- **SONG-1 (MUST):** A parent can search a music catalog by title and artist and select a track for their child.
- **SONG-2 (MUST):** A parent can set the start offset by scrubbing the track and hearing where it will begin.
- **SONG-3 (MUST):** A parent can set the clip length. Default clip length is 15 seconds. Maximum is 30 seconds.
- **SONG-4 (SHOULD):** A parent can toggle a short fade-out at the end of the clip. Default is on.
- **SONG-5 (MUST):** A parent can preview the exact clip (from the chosen offset, for the chosen length, with fade if enabled) before saving.
- **SONG-6 (MUST):** A parent can change or replace their child's song at any time before a game pack is prepared.
- **SONG-7 (SHOULD):** A parent can save one or more alternate songs and choose which is active.
- **SONG-8 (MAY):** A parent can record and attach a short spoken intro (see 5.J).
- **SONG-9 (SHOULD):** If a parent has not set a song by a manager-visible deadline, the manager is able to see the gap and set a fallback song on the child's behalf (see 5.D).

### 5.D Song Review and Moderation (Manager)

- **MOD-1 (MUST):** A manager can view every player's song assignment on the team, including offset, length, and intro.
- **MOD-2 (MUST):** Tracks known to contain explicit content MUST be visibly flagged wherever songs are listed, so a manager can review them before game day.
- **MOD-3 (MUST):** A manager can reject or replace any song assignment, including for appropriateness, and can set a fallback song for any player who has none.
- **MOD-4 (SHOULD):** A manager can require approval, so that a parent's selection is not eligible for a game pack until the manager approves it. When approval is not required, selections are eligible by default.
- **MOD-5 (SHOULD):** A parent is notified when their child's song is rejected or replaced, with the reason if one is given.

### 5.E Game Pack Preparation and Readiness

- **PACK-1 (MUST):** A manager can prepare a game pack for a team in a single action.
- **PACK-2 (MUST):** Preparing a game pack makes every eligible player's song (and intro, if present) available for playback with no network connection at game time.
- **PACK-3 (MUST):** Every clip in a prepared pack MUST play at a consistent perceived loudness, so no player's song is noticeably louder or quieter than another.
- **PACK-4 (MUST):** Preparation reports per-player readiness and clearly identifies any song that failed to prepare, so it can be fixed before the game.
- **PACK-5 (MUST):** A manager can re-prepare or refresh the pack after making changes, and can see which items changed.
- **PACK-6 (SHOULD):** A manager can prepare a pack for a specific upcoming game (tied to a specific lineup) or for the team generally.
- **PACK-7 (SHOULD):** A manager is warned if they attempt to start a game while any active player's clip is not ready.

### 5.F Game-Day Console: Manual Operation

- **PLAY-1 (MUST):** The console shows the team's active players, by default in batting order, one clearly tappable control per player, each labeled with name and jersey number.
- **PLAY-2 (MUST):** Each player control shows whether that player's clip is ready for offline playback.
- **PLAY-3 (MUST):** Tapping a player's control plays their walk-up clip (intro first if present, then the song segment).
- **PLAY-4 (MUST):** The operator can stop playback immediately, with a stop control that also applies a short fade so cutoffs are not abrupt.
- **PLAY-5 (MUST):** The console MUST be fully operable with no network connection once a pack is prepared.
- **PLAY-6 (SHOULD):** The operator can replay the current player's clip, and can manually select any player out of order.
- **PLAY-7 (SHOULD):** The console indicates which player is currently playing and how much of the clip remains.
- **PLAY-8 (MUST):** If a tapped player's clip is not ready, the console makes this obvious and does not fail silently. The operator can skip or fall back without the app becoming stuck.
- **PLAY-9 (SHOULD):** A "soundcheck" mode plays every ready clip in sequence so the operator can confirm the whole pack works before first pitch.
- **PLAY-10 (SHOULD):** The console is legible in direct sunlight and operable quickly with one hand (large targets, high contrast).

### 5.G Game-Day Console: Assisted Operation (Live Lineup)

Applies when a lineup and/or live game state is available. All of section 5.F still applies.

- **LIVE-1 (MUST):** When a batting order is present, the console can advance through it, presenting the next batter's control prominently.
- **LIVE-2 (MUST, core invariant):** Live game state MAY arm (pre-select and highlight) a player's song, but MUST NEVER automatically play it. Playing a clip always requires an explicit human trigger.
- **LIVE-3 (MUST):** Regardless of any live integration, the operator can always manually override which player is armed and can manually advance, go back, or jump to any player.
- **LIVE-4 (MUST):** Assisted operation is optional. The console MUST remain fully usable in pure manual mode if no lineup or live state is available, or if a live source stops responding mid-game.
- **LIVE-5 (SHOULD):** If live state lags behind the real game (for example a slow scorekeeper), the armed player may be stale, but this MUST only affect which control is highlighted, never cause a wrong or automatic play.
- **LIVE-6 (SHOULD):** The console indicates whether it is currently following live state, following a static lineup, or in pure manual mode, and shows when a live source has gone stale or disconnected.

### 5.H GameChanger Integration

Feasibility and mechanics are for technical design and may be constrained by what the external service exposes (see Section 9). Requirements describe desired capability.

- **GC-1 (SHOULD):** A manager can import a team's roster and batting order from GameChanger, rather than entering it manually.
- **GC-2 (SHOULD):** A manager can refresh the imported lineup before or during a game.
- **GC-3 (MAY):** During a game, the app can follow the GameChanger live feed to determine the current and on-deck batter, feeding the arming behavior in 5.G. Subject to LIVE-2 (arm only, never auto-play).
- **GC-4 (MUST, if GC-1 or GC-3 is implemented):** Imported lineup data MUST be retained locally once obtained, so that a loss of connectivity mid-game does not strand the operator. Live following degrades to manual operation on disconnect.
- **GC-5 (SHOULD):** Any GameChanger link is optional and clearly separable, so the product is complete and usable without it.

### 5.I Inning-Break and Extras

- **EXTRA-1 (MAY):** The app can play background music during breaks between innings from a designated set of tracks. This feature MAY depend on a live connection and is not required to work offline.
- **EXTRA-2 (MAY):** The console can include a small set of quick sound effects (for example a cheer or an air horn) that play on tap.
- **EXTRA-3 (MAY):** A manager can control inning-break playback (start, stop, skip) from the console without disrupting the ability to play a walk-up clip on demand.

### 5.J Intros

- **INTRO-1 (MAY):** A parent or manager can record a short spoken intro (target a few seconds) for a player.
- **INTRO-2 (MAY):** When an intro exists and is enabled, it plays immediately before the player's song on the console.
- **INTRO-3 (MAY):** Intros are included in game-pack preparation and must be available offline like songs.
- **INTRO-4 (MAY):** A text-based announcer intro is an acceptable alternative mechanism if preferred over recorded audio.

### 5.K Persistence, Reuse, and Multi-Team

- **DATA-1 (MUST):** Song assignments, rosters, and intros persist across games without re-entry.
- **DATA-2 (SHOULD):** A manager can carry a roster and its song assignments into a new season or a new game.
- **DATA-3 (SHOULD):** A player's song history is retained so a prior song can be reinstated.
- **DATA-4 (MAY):** A user who manages or parents on multiple teams can switch between them without signing in again.

---

## 6. User Flows

Each flow lists the actor, the steps, and the system's expected behavior. Requirement IDs in brackets indicate the governing requirements.

### 6.1 Manager creates a team and roster
1. Manager creates a team with a name and season. [TEAM-1]
2. Manager adds players, optionally with jersey numbers, positions, and batting order. [TEAM-2, TEAM-4]
3. System saves the roster and makes it available for invitations and configuration. [DATA-1]

### 6.2 Manager invites parents
1. Manager generates a shareable invitation. [INV-1]
2. Manager optionally pre-assigns players to parent emails. [INV-3]
3. Manager posts or sends the invitation. [INV-1]
4. System shows, per player, whether a parent has joined and whether a song is set. [INV-5]

### 6.3 Parent onboards and claims a child
1. Parent opens the invitation. [INV-1]
2. Parent signs in with a low-friction method, without creating a password. [INV-2]
3. Parent claims their child from the roster (or confirms a pre-assignment). [INV-3]
4. System grants the parent edit rights to that child only. [ROLE-2]

### 6.4 Parent sets a walk-up song
1. Parent searches for a track by title or artist. [SONG-1]
2. Parent scrubs to the drop and sets the start offset. [SONG-2]
3. Parent sets clip length (default 15s) and fade. [SONG-3, SONG-4]
4. Parent optionally records an intro. [SONG-8, INTRO-1]
5. Parent previews the exact clip. [SONG-5]
6. Parent saves. System marks the assignment set and, if approval is required, pending. [SONG-6, MOD-4]

### 6.5 Manager reviews and moderates songs
1. Manager reviews all assignments, with explicit tracks flagged. [MOD-1, MOD-2]
2. Manager approves, rejects, or replaces songs, and sets fallbacks for players with none. [MOD-3, MOD-4]
3. System notifies affected parents of rejections or replacements. [MOD-5]

### 6.6 Manager imports the lineup (optional)
1. Manager links the team to its GameChanger equivalent. [GC-1]
2. Manager imports roster and batting order, or refreshes an existing import. [GC-1, GC-2]
3. System retains the imported lineup locally. [GC-4]

### 6.7 Manager prepares the game pack
1. Manager triggers preparation for the team or a specific game. [PACK-1, PACK-6]
2. System makes all eligible clips and intros available offline, at consistent loudness. [PACK-2, PACK-3, INTRO-3]
3. System reports per-player readiness and flags any failures. [PACK-4]
4. Manager fixes flagged items and re-prepares as needed. [PACK-5]

### 6.8 Operator runs soundcheck
1. Operator opens the console and starts soundcheck. [PLAY-9]
2. System plays each ready clip in sequence. [PLAY-9]
3. Operator confirms readiness or returns to fix gaps. [PACK-4, PACK-7]

### 6.9 Game day: manual operation
1. Operator opens the console; connectivity may be absent. [PLAY-5]
2. Console shows active players in batting order, each showing ready state. [PLAY-1, PLAY-2]
3. As a player walks up, operator taps their control. [PLAY-3]
4. System plays the intro (if any) then the song segment. [PLAY-3, INTRO-2]
5. Operator stops with fade when the player is set, or lets the clip end. [PLAY-4]
6. If a clip is not ready, console shows this and operator skips or falls back. [PLAY-8]

### 6.10 Game day: assisted operation
1. A lineup and/or live state is present. [LIVE-1]
2. System arms and highlights the next or current batter. [LIVE-1, LIVE-2, GC-3]
3. Operator confirms by tapping to play; system never auto-plays. [LIVE-2]
4. Operator can override the armed player, advance, go back, or jump. [LIVE-3]
5. If live state lags or disconnects, only the highlight is affected; operator continues manually. [LIVE-4, LIVE-5]

### 6.11 Failure handling at the field
1. Network is lost. Console continues from the prepared pack. [PLAY-5, GC-4]
2. A live source stops responding. Console falls back to manual, indicating the change. [LIVE-4, LIVE-6]
3. A specific clip is missing or failed preparation. Console flags it; operator skips or plays a fallback without getting stuck. [PLAY-8]

### 6.12 Cross-game and season reuse
1. Manager starts a new game or season. [DATA-2]
2. Manager carries forward the roster and assignments, adjusting as needed. [DATA-2, DATA-3]
3. Parents update only what changed. [SONG-6]

---

## 7. Quality and Behavioral Requirements

Observable requirements, not design choices.

- **Q-1 (MUST):** Game-day playback works with the device offline once a pack is prepared. [ties PLAY-5, PACK-2]
- **Q-2 (MUST):** When the operator triggers a clip, playback begins within a short, consistent delay. Target under 1 second; this MUST hold offline.
- **Q-3 (MUST):** No prepared clip plays noticeably louder or quieter than another. [ties PACK-3]
- **Q-4 (MUST):** A parent can never see or edit another child's configuration. [ties ROLE-2]
- **Q-5 (MUST):** No live source or integration can cause a song to play automatically; a human trigger is always required. [ties LIVE-2]
- **Q-6 (SHOULD):** The console is legible in direct sunlight and usable one-handed. [ties PLAY-10]
- **Q-7 (SHOULD):** Any failure at the field (missing clip, lost network, dropped live feed) degrades gracefully and never leaves the console stuck.

---

## 8. Constraints and Assumptions

These shape technical design without prescribing it.

- **C-1:** The product MUST NOT require any new paid subscription from the user.
- **C-2:** The following are available as existing resources; how they are used is a technical-design decision, not a requirement: an active Spotify Premium subscription and an active YouTube Premium subscription.
- **C-3:** The product is for personal / club use only, at low and limited usage volume. It is not distributed commercially and does not need to scale beyond a small number of teams.
- **C-4:** The builder accepts that some capability may rely on unofficial or gray-area access to third-party services, and is comfortable with that risk given the non-commercial, limited-usage nature of the project. Where a choice exists, prefer approaches that keep any such risk contained and that fail in advance of game day rather than during it.
- **C-5:** Streaming-service reliability is assumed to be imperfect. The design MUST ensure that streaming problems cannot affect game-day playback. This is why offline readiness (Section 5.E) is a hard requirement rather than a convenience.
- **C-6:** The primary game-day device is a phone used by the operator at a field, where connectivity is often poor or absent.

---

## 9. Dependencies and Open Questions

To be resolved during or before technical design. Listed so they are not mistaken for settled requirements.

- **O-1:** GameChanger exposes no known official public interface. The feasibility, stability, and terms-of-use implications of roster import (GC-1), lineup refresh (GC-2), and live following (GC-3) must be assessed during technical design. All GameChanger requirements are conditional on that assessment, which is why they are scoped SHOULD/MAY and why the console must be complete without them (GC-5, LIVE-4).
- **O-2:** The mechanism by which a searchable catalog selection (SONG-1) becomes an offline, loudness-normalized clip (PACK-2, PACK-3) is a technical-design question. The requirement is only that both hold.
- **O-3:** Whether inning-break music (EXTRA-1) and any live catalog features are worth the added dependency is a product decision that can be deferred; they are MAY.
- **O-4:** Sign-in method (INV-2) must satisfy the no-password constraint; the specific method is for technical design.
- **O-5:** The retention and privacy handling of children's names and parent contact information should be reviewed, given the users are minors, even for a personal project.

---

## 10. Suggested Prioritization

Functional sequencing only. The engineer determines build order within this.

- **Milestone 1 (Usable console):** Manual roster entry, song assignment, game-pack preparation with offline readiness and loudness normalization, and the manual game-day console with soundcheck. Sections 5.A, 5.C (single active song), 5.E, 5.F. This alone is a working product and directly fixes the "basic functionality is broken" problem.
- **Milestone 2 (Multi-user self-service):** Invitations, parent onboarding, per-child permissions, and manager moderation. Sections 5.B, 5.D, ROLE rules. This delivers the parent-managed-songs capability.
- **Milestone 3 (Assisted operation):** Static lineup advancement and the arming model. Section 5.G.
- **Milestone 4 (GameChanger):** Roster/lineup import, then live following if feasible. Section 5.H, subject to Section 9.
- **Milestone 5 (Extras):** Intros, inning-break music, sound effects. Sections 5.I, 5.J.

---

## 11. Acceptance Criteria (Milestone 1 and core invariants)

Pass conditions for the parts that define "working."

- **AC-1:** With the device's network fully disabled, the operator opens the console for a prepared team and plays any active player's clip successfully. [Q-1, PLAY-5, PACK-2]
- **AC-2:** Triggering a clip starts playback within the target delay, offline. [Q-2]
- **AC-3:** Playing several different players' clips in a row produces no jarring volume differences. [Q-3, PACK-3]
- **AC-4:** Preparing a pack where one song cannot be resolved yields a clear per-player failure indication and does not block the rest of the pack from being ready. [PACK-4]
- **AC-5:** A parent account signed into a team can view and edit exactly one child's song configuration and cannot reach any other child's, by UI or otherwise. [Q-4, ROLE-2]
- **AC-6:** With live state connected, the console highlights the current or on-deck batter but never begins playback without a human tap; disconnecting the live source leaves the console fully operable in manual mode. [Q-5, LIVE-2, LIVE-4]
- **AC-7:** A tapped player whose clip is not ready produces a clear not-ready state and an available fallback, with no stuck or crashed console. [PLAY-8, Q-7]
