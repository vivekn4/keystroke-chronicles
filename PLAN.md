# Keystroke Chronicles — PLAN.md

**Crafted by Vivek Nair · VN** · Vanilla HTML5 + CSS + JS (single `index.html`, no build, no dependencies) · MIT

A story-driven typing RPG: you are a Leximancer — a mage whose spells are cast by
*typing words*. Six story chapters across the Ember Isles, each ending in a duel
against a corrupted boss. Original fantasy prose throughout (no copyrighted text).

## 1. The game in one paragraph

The Inkblight — a living corruption that eats written words — is spreading across
the Ember Isles, and the Great Lexicon is fading. You play **Ash**, a young
Leximancer apprentice whose master has vanished. Across 6 chapters you travel the
isles, duel corrupted foes by typing spell-words faster and cleaner than they can
hurl curses at you, unlock spells through accuracy streaks, and decide the fate of
the Lexicon. Three endings (Dawnbringer / Ashen / the secret Inkbound ending) are
earned by performance, not picked from a menu.

## 2. Architecture (where things live)

Single-file `index.html`: `<style>` block, `<canvas>` game layer, DOM overlay
layer (menus / dialogue / reports), one `<script>` block organized in sections:

| System | Owner | Notes |
|---|---|---|
| Game state machine | `Game.state` | TITLE, HOWTO, DIALOGUE, BATTLE, LEVELUP, REPORT, ENDING; PAUSE & SETTINGS are overlays on top of any state |
| Content data | `CHAPTERS[]`, `WORDS{}`, `BOSSES{}` | All prose, word lists, enemy stats as data — zero logic mixed in |
| Battle engine | `Battle` object | Word queue, enemy spawner/timers, HP, combo, accuracy, spell triggers |
| Input router | `onKey` handler | Routes keystrokes by state; battle input does per-character validation |
| Renderer | canvas 2D, `draw()` | Background gradient + stars/embers, enemies, floating words, particles, HUD; DOM overlays for menus/dialogue |
| Particles | `particles[]` pool | Fixed pool (300), pooled objects, no per-frame allocation |
| Screen shake | `shake` magnitude decay | Toggleable in settings; trauma-based (adds on hits, decays) |
| Audio | `Sfx` module | WebAudio synthesized, master gain, mute toggle; all sounds generated, no files |
| Save | `Save` module | localStorage `kc_save_v1`: chapter, level, stats, settings, best WPM/accuracy, endings unlocked |
| WPM/accuracy | `Stats` | Live WPM (correct chars / 5 / minutes), accuracy = correct / total keystrokes |

### Battle loop (the core mechanic)
1. A spell-word appears above the player; typing it character-by-character charges
   the spell. Completing the word fires a bolt: damage = `word.length × power × comboMult`.
2. Combo: each fully correct word increments combo; a wrong keystroke breaks combo
   to 0 and deals 2 self-damage ("backlash") — plus feeds the enemy's corruption
   meter slightly.
3. Enemies attack on timers: a "curse word" spawns at the enemy and drifts toward
   the player. Typing it fully before it arrives destroys it (+small damage to
   enemy). If it reaches the player, it deals its damage.
4. Spells (unlocked by accuracy streaks, one active at a time via 1/2/3 keys):
   - **Emberbolt** (start): the basic attack, always available.
   - **Fireball** (10-combo): next word deals 2.5× damage with a big burst.
   - **Frost Nova** (20-combo): freezes all enemy timers for 5 s.
   - **Mend** (chapter accuracy ≥ 95% at unlock check): heals 25% max HP, once per battle.
5. Boss fights: instead of curse words, the boss channels a **passage** (3–5 original
   sentences) under a countdown timer. Finish the passage in time = massive damage
   + stun; timer expiry = heavy hit to the player. Between channels, normal
   curse-word skirmishing continues.
6. Win: enemy HP → 0. Lose: player HP → 0 (can retry the battle; no permadeath).

### Chapter structure
Each chapter: 2–4 dialogue beats (typewriter text, advance with Space/Enter) →
1–2 normal battles → boss duel → battle report card (WPM, accuracy, max combo,
time) → level-up choice (pick 1 of 3: +Power, +Vitality, +Focus) → save.

### Endings
- **Dawnbringer** — campaign cleared with average accuracy ≥ 90%: the Lexicon is restored.
- **Ashen** — campaign cleared below 90%: victory, but the isles bear scars.
- **Inkbound (secret)** — clear all bosses without losing any battle AND land a
  30+ combo at least once: Ash merges with the Lexicon itself.

## 3. Word content (original, per-chapter themed)

- Ch1 Ashes of the Scriptorium (ember/ruin words): ~40 words
- Ch2 The Drowned Library (water/ink words): ~40 words
- Ch3 Gale of the Shattered Peaks (wind/storm words): ~40 words
- Ch4 The Gilded Labyrinth (gold/maze words): ~40 words
- Ch5 The Hollow Market (shadow/trade words): ~40 words
- Ch6 Heart of the Inkblight (void/corruption words): ~40 words
- Boss passages: 1 original 3–5 sentence passage per boss (6 total), written fresh.
- Curse words reuse chapter lists (shorter words preferred for curses).

## 4. Likely bugs / problems + mitigations

| Risk | Mitigation |
|---|---|
| Keystroke handler misses keys during heavy particle frames | Input handled on `keydown` events (not per-frame polling); particles are cosmetic only |
| Backspace handling feels wrong | Backspace moves the caret back within the current word; already-fired words are locked (can't edit history) — standard typing-test behavior |
| WPM inflated by pausing mid-battle | Battle timer pauses with the game (pause overlay stops the clock); WPM uses active typing time only |
| Long words + small screens overflow | Canvas scales to window; words wrap and font size clamps; tested at 1280×720 and 390×844 |
| Timer drift makes boss countdown unfair | Boss countdown uses accumulated `dt` from rAF, not wall clock; pause freezes it |
| localStorage unavailable (private mode) | `Save` wraps in try/catch; game runs with in-memory fallback, warns once |
| Audio autoplay blocked | AudioContext resumes on first user gesture (title screen click/keypress) |
| Particle pool exhaustion on big bursts | Fixed pool of 300; spawn functions no-op when exhausted — visuals degrade, gameplay never breaks |
| Text-speed setting vs typewriter | Setting scales typewriter chars/sec; Space/Enter always skips to full text instantly |
| Unwinnable difficulty spikes | Difficulty setting scales enemy HP (±25%), curse timers (±30%), word length pools; playtested per chapter |

## 5. UI/UX thinking (including SFX/VFX)

- **Readability first:** current target word is large, centered, high-contrast;
  typed characters turn gold, the next character pulses gently. Enemy curse words
  are red-tinted and drift with a subtle wobble — threat is readable at a glance.
- **Maximalist but disciplined VFX:** ember particle bursts on spell hits,
  rune-ring shockwaves on combo milestones (10/20/30), red vignette flash + shake
  on player damage, green-gold glow on heal, slow-mo white flash on boss kill.
  All particles pooled; shake is trauma-based and toggleable.
- **SFX (all WebAudio, unique to this game):** mechanical key tick, spell charge
  riser, fireball whoosh-crack, frost chime, heal shimmer, combo milestone arps,
  player hurt thud, enemy hit crunch, level-up fanfare, victory jingle, defeat
  low dirge, UI click/hover blips, boss channel warning klaxon.
- **VN signature:** small "VN" mark in the title-screen corner and README.
- **Menus:** Title (New Game / Continue / How to Play), Pause (Resume / Restart
  battle / Mute / Quit to title), Settings (difficulty Scribe/Leximancer/Archmage,
  screen-shake on/off, text speed slow/normal/fast, mute).
- **Battle report card** after every fight: WPM, accuracy, max combo, time, damage
  dealt/taken — the "typing test" payoff the genre promises.
- **How to Play** teaches in 4 beats with a live practice word (no fail state).

## 6. QA plan

- Node syntax check on extracted JS; headless Chromium playthrough: title →
  new game → dialogue advance → chapter 1 battle (scripted typing: all-correct,
  all-wrong, mixed) → verify win path, lose path (take hits), pause/resume,
  settings toggles, save/continue across reload, WPM/accuracy sanity.
- Zero console errors across the scripted run. Screenshot captured from the real
  running game (battle mid-action) → `assets/screenshot.png`.
