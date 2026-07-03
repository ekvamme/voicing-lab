# Voicing Lab — Dev Notes

A single-file interactive tool for studying **jazz voice leading through inversions**: how a
pianist "walks" a chord progression by choosing the inversion of each chord that holds common
tones and moves the other voices the shortest distance, so the hand stays in one position.

This file is the project context. If you're continuing in **Claude Cowork**, point a session at
this folder and read this file first — it describes what exists, how the code is laid out, and
what to build next.

---

## Files
- `voicing-lab.html` — the entire app. No build step, no dependencies to install. Open it in a
  browser to run. Loads Tone.js (audio) from cdnjs, three Google Fonts from a `<link>`, and
  (lazily, on first sound) Salamander piano samples from tonejs.github.io — synth fallback if offline.
- `archive/voicing-lab-ipad.html` — retired iPad-frame fork (engine had duplicated + drifted).
  The main file is now touch-friendly (`pointer:coarse` CSS); don't extend the archived copy.

## How to run
Double-click `voicing-lab.html`, or in Cowork ask to open it in the default browser. Audio starts
on the first click/tap (browser autoplay rule), handled via `Tone.start()`.

---

## What it does today
- **Three progressions:** ii–V–I, I–vi–ii–V turnaround, and a 12-bar jazz blues (all in C).
- **Two voicing modes:**
  - *Smooth (inversions)* — each chord is voice-led from the previous one (default).
  - *Block (root position)* — every chord reset to root position, for contrast.
- **Voice-leading graph** — vertical axis = pitch, each column = a chord, lines connect each voice
  between chords. Line style encodes motion: held common tone / step / leap. Inversion name is
  labelled under every chord.
- **Hand-travel meter** — total semitones of movement for smooth vs. block, shown side by side
  (e.g. ii–V–I is 6 vs. 48). This number is the core "why" of the tool.
- **Keyboard** — shows the current chord's hand shape; common tones held from the previous chord
  get a dashed-gold outline.
- **Walking bass** — optional layer, off by default (root, 3rd, 5th, chromatic approach on beat 4).
- **Transport** — play/stop (loops), prev/next chord stepping, tempo 50–180 BPM. Arrow keys
  step chords; Space toggles play.
- **Starting-inversion selector** — anchor chord 1 on root/1st/2nd/3rd; the whole path re-solves.
  (Total travel is inversion-invariant — cost depends only on pitch classes — but the inversion
  sequence changes, which is the lesson.) Ignored/dimmed in block mode.
- **Guide-tone view** — layer toggle hiding root & 5th; graph, keyboard, and audio show only the
  3rd/7th skeleton.
- **Key selector** — transposes any progression to all 12 keys with functional spelling
  (letter-distance transposition; gnarly roots like B# get respelled).
- **Loop wrap** — faint stubs after the last chord / before the first show the last→first motion.
- **Audio** — real piano (Salamander samples via Tone.Sampler), triangle-synth fallback while
  loading or offline.
- **Insight panel** — per-chord explanation that adapts to mode (which inversion, how many common
  tones held, biggest single move).
- **Single-screen layout (v4.3–4.5)** — everything visible at once, top to bottom:
  **grand-staff sheet music → voice-leading graph (chords with inversion labels) →
  one keyboard dedicated to the left-hand chord voicing** (the quick reference for
  the walking pattern: role colors, held fingers dashed gold) → insight panel.
  All panels advance in sync with the transport.
  Retired along the way: the Layers group (walking-bass & guide-tone toggles — code
  paths remain but `showBass`/`guideTones` are const false), the melody toggle
  (melody auto-plays when the tune has one), the Keys-show seg, the hand-shape strip
  (v4.4, removed in v4.5 — one keyboard only), and the improv-scale keyboard view
  (`SCALES`/`lightScale()`/`scaleHint()` remain in the file, dormant, if the scale
  view ever earns its way back).
  - *Sheet music* — hand-drawn SVG grand staff: treble = melody (when the tune has one),
    bass = voice-led chord stacks in role colors with duration-accurate note values
    (+ walking-bass quarters when that layer is on), chord symbols at their true beat,
    key/time signatures, accidentals, beams, auto rests.
  - *Keys show* seg (was the View switcher) — toggles what the keyboard lights:
    **Hand shape** (chord voicing, held common tones dashed gold) or **Improv scale**
    (chord-scale tinted, guide tones dashed, avoid notes dotted, optional blues overlay
    with auto hint text naming the next chord's target 3rd).
- **Melody engine** — per-song melodies as `[bar][{b,d,m}]` (beat, duration-in-beats, midi),
  transposed with the key selector, played via the transport and when stepping bars.
- **Beat-accurate harmony** — chords are events with a `beats` duration (spec field, default 4),
  not one-per-bar. Each event carries `start` (absolute beat), `bar`, `beat`; `BEATMAP` maps
  every beat of the form to its event. Chord symbols and bass-clef stacks sit at their real
  beat position in the measure, stack note-values reflect duration (whole/half/quarter), the
  highlight band spans only the chord's beats, playback strikes chords exactly where they fall
  and holds them for their true length, and the walking bass adapts (4→R-3-5-approach,
  3→R-3-approach, 2→R-approach, 1→R). Qualities now include **7♯5** (whole-tone in scale view).
- **Sweet Georgia Brown (1925, public domain)** — first full sheet-music tune: 32 bars,
  A–B–A–C, dominants around the circle of fifths, with 6th/m6 chords and mid-bar changes
  (bars 14, 24, 30, 31). **Melody & changes are machine-read from the Nottingham Music
  Database ABC transcription** (abcnotation.com → abc.sourceforge.net/NMD, reelsd-g.txt
  tune #63, "S:Trad", key G) — plain-text notation parsed programmatically with
  `/tmp/abcparse.js`-style code, eliminating visual transcription error entirely. This is
  the tune as commonly played. Earlier attempts read the 1925 Remick plates by eye (History
  Harvest 909×1200, then USC Tin Pan Alley 3804×5023 IIIF: item 4929, pages 4926–4927,
  regions via `/iiif/2/tinpanalley:4926/<x>,<y>,<w>,<h>/full/0/default.jpg`) — image-based
  reading proved unreliable for note-level accuracy and should not be repeated; prefer
  machine-readable sources (ABC/MusicXML/**kern) for all future tunes.
  Known simplification: cross-bar ties are re-struck at the barline.

---

## Code map (inside `voicing-lab.html`, all in one `<script>`)
- **Music engine**
  - `spell(letter, pc)` — proper enharmonic spelling (so Cmaj7's 7th is B, F7's is E♭).
  - `chordRoot(root, quality, oct)` — root-position 4-note seventh chord; each note carries
    `{role 0..3 = R/3/5/7, pc, name, midi}`.
  - `nearestMidiForPc(pc, ref)` — places a pitch class in the octave nearest a reference note.
  - `voiceLead(prevVoicing, tones)` — the heart of it. Brute-forces all 24 assignments of the new
    chord's four tones to the four voices and keeps the one with least total movement.
  - `invIndex(voicing)` — inversion (0–3) from whichever chord tone is lowest.
  - `invert(voicing,k)` — k-th inversion of a root-position voicing (raises bottom k tones an octave).
  - `KEYS` / `transposeRoot(root,K)` — key transposition with functional spelling.
  - Qualities: maj7, m7, dom7, m7♭5, **6, m6** (`QUAL`/`QLAB`/`QLET`; 6th chords use letter step 5).
- **Notation:** `spellInKey(midi,key)` + `KEYSIG`/`keyAlteration` (functional spelling against the
  key signature), `buildSheet()` (grand staff SVG) + `updateSheetBand()` (cheap bar-highlight move).
- **Scale view:** `SCALES` (chord-scale map), `lightScale()`, `scaleHint()`; `applyView()` switches
  the Fingers/Sheet/Scale stages.
- **Melody:** `MELODIES`/`SONGKEY` registries filled when a song loads; `rebuild()` derives
  `MELODY` (transposed) and `EFFKEY`; transport schedules per-bar melody events; `playBarMelody()`
  covers manual stepping.
  - `walkingBass(rootName, nextRoot)` — the optional bass line.
- **Data:** `SPECS` (built-in progressions) + `IMPORTED` (Real Book songs) →
  `computeProgression(specs, mode, startInv)` builds voicings; `travelOf(specs, mode, startInv)`
  totals the movement.
- **Render:** `buildGraph()` (full SVG, only on prog/mode/key/anchor change) +
  `updateActiveBand()` (cheap per-bar highlight move), `renderKeyboard()` + `lightChord()`,
  `setHud()` / `updateInsight()` (text). `cssVar()` caches computed CSS variables.
- **Audio:** `initAudio()` (synths + lazy piano Sampler), `chordInstr()`/`bassInstr()` pick
  piano when loaded, `playChord()`, `tapNote()`, `startPlay()` / `stopPlay()` (Tone.Transport).
- **State:** `progIdx`, `mode`, `startInv`, `keyIdx`, `guideTones`, `curBar`, `playing`,
  `showBass`; `CH` is the current computed progression, `CURSPECS` the (possibly transposed)
  specs, `TRAVEL` the cached travel totals. `rebuild()` recomputes after any of those change.

## Design tokens (CSS `:root`)
Warm espresso/brass jazz-club palette. Voice role colors: `--v0` root (clay), `--v1` 3rd (amber),
`--v2` 5th (teal), `--v3` 7th (indigo). Motion colors: `--hold` gold, `--step` green, `--leap`
orange. Fonts: Fraunces (display), Space Grotesk (UI), JetBrains Mono (data/labels).

---

## Deliberate simplifications (good to know before extending)
- The first chord of each loop is anchored (user-selectable inversion); everything else is
  voice-led from it. The loop wrap (last → first) is not counted in hand-travel; it is drawn
  as faint stubs only.
- Voices may theoretically cross, though minimal-motion matching almost never does.
- THEORY panel texts describe songs in their written key; they don't re-render when transposed.

## Direction (set 2026-06-23)
**Phase 1 — optimize as a teaching tool first.** Focus on the pedagogy and UX of the
voice-leading lessons before any packaging work. **Phase 2 — then make it a launchable app**
(leading candidate: Capacitor wrapper for a real iPad `.app`; PWA as a lighter interim step).
Do not start app-packaging work until the teaching tool is where we want it.

**Skill-level tiers (toggle between them).** The teaching tool should adapt to player level
via a difficulty toggle:
- *Beginner* — triads (not 7th chords), shorter/simpler progressions, slower default tempo,
  note names shown, heavy visual scaffolding, the "math" (hand-travel numbers, brute-force
  detail) hidden behind plain-language hints.
- *Medium* — current behavior: 7th chords, ii–V–I / turnaround / blues, inversions, the
  hand-travel meter and common-tone highlighting.
- *Advanced* — guide-tone view, drop-2/drop-3/rootless/shell voicings, tritone subs &
  secondary dominants, key transposition, custom progression input; hints minimized.
Many roadmap items below are really the Advanced tier; a new Beginner tier is the main net-new work.

## Roadmap / next ideas (rough priority)
1. **Beginner tier** (see skill-level tiers above) — the main net-new work.
2. **Drop-2 / drop-3 / rootless / shell voicing modes** — compare real named hand shapes.
3. **Melody-on-top lock** — pin the soprano line to a chosen note and re-voice underneath.
4. **Custom progression input** — type chord symbols (e.g. "Cmaj7 A7 Dm7 G7").
5. **Print/export** — a clean static SVG of the voice-leading graph for practice sheets.
6. **Tritone subs / secondary dominants** — show how reharmonization changes the voice leading.

## Changelog
- v1 — block-chord tool with walking bass as the focus.
- v2 — refocused on voice leading through inversions; added the pitch-height graph,
  smart inversion picking, hand-travel meter, common-tone highlighting; bass demoted to a toggle.
- v3 (2026-07-02) — perf pass (cached travel totals + CSS vars, per-bar highlight moves
  instead of full SVG rebuilds, debounced resize); retired the iPad fork (main file now
  touch-friendly); added starting-inversion selector, guide-tone view, 12-key transposition,
  loop-wrap stubs, arrow-key/Space transport, real piano samples with synth fallback.
- v4 (2026-07-02) — play-along release: three-view switcher (Fingers / Sheet music /
  Scale), hand-drawn grand-staff renderer, melody data model + playback, improv scale view
  with chord-scales + blues overlay, 6th/m6 chord qualities, Sweet Georgia Brown (public
  domain) as the first full tune with melody. See PUBLIC_DOMAIN_STANDARDS.md for which tunes
  can legally get the same treatment.
- v4.1 (current, 2026-07-02) — beat-accurate harmony: chords carry durations and change
  mid-measure exactly as the 1925 plates print them (D7→D7♯5 in bars 12/14, G6→Em7 in 15,
  B7→Em6 in 24, G6→B7 turnaround in 32); added the 7♯5 quality; melody bars 12/14 corrected
  to A♯ (the ♯5 color the plates harmonize). SGB is now 37 chord events over 32 measures.
