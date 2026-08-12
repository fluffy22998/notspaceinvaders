# Rocket Game — Development Outline
**Working title:** Not Space Invaders
**Target players:** Ages 5–7
**Platform:** iOS + Android
**Engine:** Unity, **2D** (URP 2D Renderer)

---

## Engine & Project Setup Decisions

**2D, not 3D.** Everything about this game — side-scrolling plane, sprite-based rocket, flat hazard shapes, high-contrast silhouette readability — is a 2D problem. 3D would add asset cost (modeling, rigging, lighting) for zero gameplay benefit, and would make the "chunky, high-contrast, low-clutter" readability goal from the design harder to hit, not easier. Use Sprite Shape / 2D Animation package for any squash-stretch on the rocket, not bones.

**Core packages:**
- Universal Render Pipeline (2D Renderer) — lets you use 2D lights for polish later without a pipeline migration
- New Input System — needed for clean touch handling across iOS/Android
- Cinemachine (2D confirm/shake) — for the "speed via camera, not velocity" trick in beat 4
- Addressables — you'll have 100+ small art assets (parts, stickers); don't ship them all in one scene bundle
- TextMeshPro — even with a non-reading-first UI, you'll want it for parent-facing screens

**Repo structure for Claude Code:** Given your existing CLAUDE.md + Unity MCP pipeline, I'd set up `/Assets/_Game/{Scripts,Art,Audio,Prefabs,ScriptableObjects}` with a `PHASES.md` alongside CLAUDE.md that tracks which phase/subphase is active — mirrors what you did on the casino sim. Each phase below is scoped to be a clean subagent handoff with a testable "done" state, since that's the methodology you're already running.

---

## Phase 0 — Feel Prototype (Grey-box)
*Goal: prove the flying feels good before any art exists. This phase gates everything else — do not proceed until it passes the kid test.*

**0.1 — Touch input → target Y**
- Absolute Y-mapping (finger height on screen = target height on screen), not relative drag
- Touch-loss behavior: hold position for 1s, then gentle downward drift
- Input System touch/mouse abstraction so it works in-editor and on-device identically

**0.2 — Spring-damped rocket movement**
- Rocket accelerates toward target Y with high damping (minimal/no overshoot — see age-band notes)
- Velocity-based tilt (rotation proportional to dY/dt, clamped)
- Expose damping, max speed, tilt clamp as ScriptableObject values for live tuning

**0.3 — Scroll speed against reaction-time budget**
- Constant right-to-left world scroll (rocket fixed at ~25% screen width)
- Hard constraint: any obstacle spawned at the right edge must take 1.2–1.5s to reach the rocket's X position at target scroll speed
- Build a debug overlay showing seconds-to-impact for the nearest hazard, for tuning

**0.4 — Grey-box test scene**
- Rectangle rocket, rectangle hazards, no art, no rewards
- Playable end-to-end for 60 seconds on a real device

**Exit criteria:** Put an iPad in front of an actual 5-year-old and an actual 7-year-old. If the rocket "feels like it's disobeying" or either kid loses the rocket visually against the background, stop and re-tune before Phase 1.

---

## Phase 1 — Hazard & World Systems
*Goal: the scrolling world with hazards, but still no reward loop.*

**1.1 — Scrolling background system**
- Parallax layers, background capped at ~30% visual weight vs. foreground (readability constraint from design)
- Seamless tiling/looping per environment (Earth first)

**1.2 — Hazard spawner**
- Object-pooled hazard spawning (mobile perf — no runtime instantiate/destroy churn)
- ScriptableObject-defined hazard "patterns" (gap position, width, movement type) rather than hardcoded — this is what lets you author beats later without code changes

**1.3 — Hazard behaviors**
- Static gap hazards (asteroid pairs with a passable gap)
- One dynamic type: slow pulsing gate (open/close) OR sine-wave mover — pick one for v1, per the "one interesting hazard, not clutter" rule
- Max simultaneous on-screen hazards: 1 for early route nodes, up to 3 by late-game — never more

**1.4 — Collision**
- Circle/capsule colliders sized generously smaller than sprite bounds (forgiving hitboxes — critical for this age group)
- Hit response: lose a shield (see 1.5), brief invincibility + flash, knockback optional (test — may disorient)

**1.5 — Shield/lives system**
- 3 shields per run, visually shown as icons (not a number)
- Losing all 3 ends the run early → still routes to reward screen with partial rewards (never a "you lost" state)

**Exit criteria:** A full 60s run is playable with real hazards, shields work, no reward loop yet — round just ends and logs to console.

---

## Phase 2 — Collectibles & Round Structure
*Goal: the core loop — weave for stars, not just survive.*

**2.1 — Star/crystal collectibles**
- Placed in ScriptableObject patterns alongside hazards, biased toward risky lanes
- Pooled spawn/despawn, simple collect VFX + sound stub

**2.2 — Beat system (round composition)**
- Round = sequence of ~15s "beats" with a defined pattern-density curve + 2s calm buffer between beats
- Beat data as ScriptableObjects so you (or Claude) can author new rounds without touching code: `BeatDefinition { duration, hazardPatternSet, starPatternSet, calmBufferAfter }`
- Beat 4 "fast" feeling achieved via camera FOV/shake/particle trail, NOT increased scroll speed (holds the reaction-time budget from Phase 0)

**2.3 — Round timer & docking sequence**
- 60s countdown (visual, not numeric-heavy — a filling bar works better than a ticking number for this age)
- Scripted, always-succeeds landing/docking animation at round end — this is the guaranteed dopamine hit, independent of performance
- Wire round-end → star tally → (stub) reward screen

**2.4 — Adaptive difficulty (invisible)**
- Track hits-per-run locally; widen next run's gap tolerance ~15% if hit count is high, tighten slightly if a run was hitless
- Never surfaced to the player — purely a pattern-selection weight

**Exit criteria:** A full round plays: warm-up → weave → single-hazard-focus → dash → docking → star tally. Feels like a complete "trip," not a timer running out.

---

## Phase 3 — Progression & Rewards
*Goal: the meta-loop that brings a kid back tomorrow.*

**3.1 — Rocket parts system**
- 4 slots (nose, body, fins, exhaust/decal), ~6 options each at launch (24 assets → 1,296 combos)
- ScriptableObject part definitions + a simple part-compositor that layers sprites on the rocket prefab at runtime

**3.2 — Visual (non-numeric) unlock progress**
- 20-slot star tray per unlock target, filling in as icons — no fractions, no big numbers
- Persistent across runs (local save — see Phase 6 on data)

**3.3 — Route map (Earth → Moon → Mars → Belt → Jupiter)**
- Simple node-map scene, tap to select destination
- Each node = a set of beat/hazard/star pattern difficulty tiers + its own background/palette
- Unlocking a node reveals new part options tied to that theme

**3.4 — Naming ceremony**
- Three-wheel word-card picker (adjective / noun / [optional 3rd, e.g. emoji]), illustrated + spoken aloud, no keyboard
- Name displays on hull in-game and on save file selector

**3.5 — Sticker book**
- Free-placement canvas (no grid snap, no invalid-placement state), scenes = planets/stations unlocked via route progress
- Big single undo button; stickers earned from runs same as parts

**Exit criteria:** A kid can complete a round, watch their star tray fill, occasionally pop a new part or sticker, and see it appear on their named rocket. Full loop, one destination's worth of content.

---

## Phase 4 — Non-Reader UX & Audio
*Goal: nothing in the game is gated behind reading.*

**4.1 — Icon-first UI pass**
- Every button: icon primary, text secondary (supports the 7-year-olds who are reading, doesn't block the 5-year-olds who aren't)
- Large touch targets sized for kid finger accuracy + tablet-first layout, letterboxed to phone

**4.2 — Voice-over system**
- Every menu action, reward, and instruction has a spoken line (VO stub with placeholder TTS is fine for now, real VO later)
- Central `VoiceLine` ScriptableObject registry so lines are swappable without code changes

**4.3 — Silent-mode design**
- Assume many kids play muted: every audio cue (hit, collect, unlock) needs a paired visual cue that carries the same information alone
- Haptics on collect/hit/unlock for iOS + Android (Input System / Gamepad haptics or platform-specific)

**4.4 — "Show, don't tell" tutorial**
- Ghost-finger demo on first launch: animated cursor drags up/down, no text/dialog required
- No modal tutorial popups after that — teach through the warm-up beat's generous spacing instead

**Exit criteria:** Mute the device, hand it to a non-reading 5-year-old, and see if they can get through menu → round → reward → back to menu with zero adult help.

---

## Phase 5 — Mobile Platform & Performance
*Goal: this needs to run well on the hand-me-down iPad or older Android phone a 5-year-old actually uses.*

**5.1 — Orientation & safe area**
- Lock to landscape (natural two-hand or propped-tablet hold for this genre)
- Safe-area-aware UI anchoring for notches/home indicators on both platforms

**5.2 — Performance targets**
- Test on a *deliberately low-end* device profile, not your dev phone — this age group's device is very often a 3+ year old hand-me-down
- Object pooling audit (Phase 1/2 systems), texture atlas for hazards/collectibles, sprite draw call batching
- Target 60fps on mid device, hard floor 30fps on low-end

**5.3 — Build pipeline**
- iOS: Xcode project settings, App Store Connect setup, TestFlight for kid-testing builds
- Android: keystore setup, internal testing track on Play Console
- CI consideration: if you want Claude Code driving builds, a simple GitHub Actions + Unity Cloud Build (or self-hosted) pipeline is worth setting up in this phase, not bolted on later

**5.4 — Accessibility pass**
- Colorblind-safe hazard/star distinction (shape + color, never color alone — asteroids vs. stars need a silhouette difference, not just a hue difference)
- Reduced-motion option for camera shake in beat 4 (some young kids get queasy from screen shake on tablets)

---

## Phase 6 — Data, Privacy & Compliance
*Goal: this is a kids' app — this phase is not optional and not late-stage.*

**6.1 — Local-only save data**
- No account creation, no login, no cloud sync requiring PII — local device save (JSON or PlayerPrefs-backed) is sufficient and is the compliance-safe default
- If you ever want cross-device sync later, that's a distinct COPPA-scoped decision, not a default-on feature

**6.2 — Parental gate**
- Required by Apple's Kids Category and good practice generally before any settings/external-link/purchase screen (even if no IAP now, build the gate component early — you'll want it for a privacy-policy link at minimum)
- Simple math-problem or press-and-hold gate, not a button a 5-year-old could pass

**6.3 — No third-party trackers, no ad SDKs**
- Confirm no analytics SDK that fingerprints or tracks across apps; if you want any telemetry, it needs to be anonymous, aggregate, on-device-first
- This also simplifies App Tracking Transparency — you likely don't need to prompt for it at all if you're not tracking

**6.4 — Privacy policy + store listing**
- Apple Kids Category and Google Families program both have specific listing/content requirements — worth reading both policies fully before submission, not just before you hit "publish"

---

## Phase 7 — Content Pipeline & Art
*Goal: this is where your actual budget goes — plan it as its own phase, not an afterthought.*

**7.1 — Art style lock**
- High-contrast, chunky silhouettes, flat shading reads best for this age and this screen size — lock the style with a handful of final-quality assets before mass-producing
- Establish the sprite spec (canvas size, pivot points, layering order) so parts snap together cleanly in 3.1's compositor

**7.2 — Asset list & batching**
- Full inventory: 24 rocket parts (v1), hazard set per route node (~3-5 nodes × few hazard types), star/collectible variants, sticker set, background per node, UI iconset
- This is 100+ assets — budget/scope explicitly here rather than discovering the gap mid-build

**7.3 — Audio/VO production**
- SFX pass (hit, collect, unlock, docking, menu)
- VO recording once script (4.2) is locked — kid-friendly voice casting matters more than most other choices here

---

## Phase 8 — Playtesting & Iteration
*Goal: this is a recurring phase, not a final one — loop back here after Phase 0, 2, and 4 at minimum.*

**8.1 — Structured kid sessions**
- 5-year-old and 7-year-old testers, separately, recorded (with consent) for hand/finger behavior — you're watching where they actually touch, not where you assume
- Watch for: losing the rocket visually, confusion at round-start, whether they understand the shield-loss without being told

**8.2 — Parent-side testing**
- A parent should be able to hand over the device and walk away for the full round length without being needed — that's the real usability bar for this age

**8.3 — Tuning pass**
- Feed findings back into Phase 0/1 ScriptableObject values (damping, scroll speed, hazard density) before content lock

---

## Phase 9 — Polish & Submission

**9.1 — Full route content pass** — all nodes populated, difficulty curve reviewed end-to-end
**9.2 — Store assets** — screenshots, preview video, kids-category-compliant listing copy
**9.3 — Age rating + store review prep** — both Apple and Google have distinct kids-content review criteria; expect at least one review round-trip
**9.4 — Soft launch** — Android internal testing track / iOS TestFlight external group before full release, watching crash/perf telemetry on real low-end devices

---

## What you were missing (flagged separately, since these are easy to skip)

- **Parental gate** — genuinely easy to forget until an app reviewer flags it. Build it in Phase 6, not Phase 9.
- **Muted-audio design** — a huge fraction of kid play sessions are silent (car rides, quiet time). Every audio-dependent cue needs a visual twin.
- **Reduced-motion toggle** — camera shake + young kids + tablets is a real motion-sickness combo worth a toggle.
- **Colorblind-safe hazard/star distinction** — shape, not just color.
- **Low-end device testing** — the target user's device skews old. Test on your worst device, not your best.
- **Landscape-only lock** — worth deciding explicitly now, since it affects every UI layout decision downstream.
- **No login/account** — resist any temptation to add one; it's a compliance and UX liability with zero upside for this audience.

---

## Suggested build order for Claude Code sessions

Given your existing multi-agent phased workflow: Phase 0 and 1 are good single-agent, tight-loop work (fast iteration, lots of live tuning). Phase 2–3 are good candidates for the subagent-per-system split you've used before (beat system / progression system / sticker book as separate subagents against shared ScriptableObject contracts). Phase 4–6 lean toward Sonnet for UI/data plumbing; save Opus for Phase 0's feel-tuning and Phase 8's playtesting-driven redesign work, where judgment calls matter more than throughput.
