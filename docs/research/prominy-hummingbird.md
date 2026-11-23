# Prominy Hummingbird: strumming and MIDI mapping (primary-source findings)

Only source used: the official Prominy Hummingbird User Manual (PDF, 136 pages).
URL: <https://prominy.com/data/pdf/manual/en/Hummingbird/Hummingbird_user_manual.pdf>
Cited below as **[M p.N]**, the PDF page number (matches the printed page number).
Text was extracted and read directly, then paraphrased.
Pages 61 onward ("Mapping & Key Range", chord recognition intervals) are mostly images. Their tables were not read.

## 1. Does the instrument strum itself?

**Yes, for chord instruments.** One trigger plays a strum. Individual note events are not required.

- Chord instruments (major, minor, 7th, add9, sus4, user chords and so on) play a whole chord from one key.
  - Strum Keys play down/up strokes on all strings or on a subset. [M p.16, p.22]
  - The Normal Key range, E3–C7, is also described as the main range for "chord strumming". [M p.16]
- Chords are either **sampled** (real recordings) or **emulated** (built from single-note samples, with a configurable inter-string time lag, "Stroke Speed", and "Extra Strum Noise"). [M p.7, p.34–35]
- Chord Mode is Hybrid or Emulated, selected by CC #24. Hybrid uses sampled chords when available. [M p.34]
- Strum settings (per-string on/off, velocity rate, stroke type, direction) apply **only to emulated chords**. Editing them forces emulated chords even in Hybrid mode. [M p.41]
- Stroke direction (down/up) is automatic by default, from beat position. It works only while the host sequencer is playing. [M p.36–37]
  - CC #58 selects the mode: auto, forced alternate, down only, up only.
  - CC #57 sets the auto-mode resolution: 8th, 8th triplet, 16th, 16th triplet, 32nd.
- The manual documents no strum-pattern sequencer or pattern data. A pattern is made by the order and timing of Strum Key or Normal Key notes plus the direction settings.
- Single-note instruments do **not** strum. The Strum Keys then act as "repeat same note" keys. [M p.22]
- Arpeggio Keys play one string of the current chord per key. The manual says this is "not a traditional arpeggiator". [M p.23]

## 2. Documented MIDI note map (default)

The manual gives A#7 = MIDI 118 for the string/chord-shape key switch [M p.25] and E1 = MIDI 40 in a CC #44 example [M p.40]. These values establish the manual's octave naming; other numeric values derived from pitch names must be identified as conversions, not quoted manual mappings.

| Function | Default keys | Source |
|---|---|---|
| Instrument Select key switches (articulation / chord type) | C-2 to E1 (MIDI 0–40, converted from the manual's note names) | M p.16, p.18, p.40 |
| Play Keys | F1 hold (finger-release/fret noise), F#1 hold (Gliss Down), G1 hold (fret/position-change noise), G#1 picking noise, A1 muted brush noise, A#1 repeat same note | M p.24 |
| Strum Keys 1–8 | C2 full down, C#2 full up, D2 mute down, D#2 mute up, F#2 strings 4–6 down, G#2 strings 4–6 up, A#2 strings 1–3 down, C#3 strings 1–3 up | M p.22 |
| Arpeggio Keys | E2, F2, G2, A2, B2, C3 = strings 6 to 1 | M p.16, p.23 |
| Normal Key | E3–C7 | M p.16 |
| String / chord-shape select | F7 (113) = string 6 / shape 1, F#7 (114) = string 5 / shape 2, G7 (115) = string 4 / shape 3, G#7 (116) = string 3 / shape 4, A7 (117) = string 2 / shape 5, A#7 (118) = string 1 | M p.25, p.54 |
| String Skip | C8 (120) = string 6 up to F8 (125) = string 1 | M p.54 |

- Muted Strum Keys 3 and 4 strike strings 4–6 muted and **stop** strings 1–3. Keys 5–8 neither stop nor strum the other strings. [M p.22]
- Strum Key, Play Key and Instrument Select assignments are customizable in the Options tab. Instrument Select is also configurable by CC #44–46. [M p.22, p.24, p.38–40, p.54]
- Default Instrument Select assignments are on [M p.18]. Examples: E0 = major, D#0 = minor, G0 = 7th, D0 = add9, F0 = sus4, E1 = single note no legato. Some instruments appear unassigned by default, and the extracted text does not show which. Check the picture before relying on it.
- **Chord recognition.** Playing a chord's notes inside the key-switch range selects that chord instrument. [M p.26, p.38] The intervals are on [M p.103 onward], images.

## 3. Articulations / MIDI CC (all from the manual)

| CC | Function | Values | Page |
|---|---|---|---|
| 1 | Mute / picking noise (mod-wheel mode) | mute above 31, picking noise at 127 by default | M p.20, p.52–53 |
| 3 | Fast / slow stroke (chord instruments) | 0–63 fast, 64–127 slow | M p.21 |
| 4 | Gliss Down speed | 0–42 fast, 43–85 mid, 86–127 slow | M p.21 |
| 15 | Forced hammer-on / pull-off (Emulated mode only) | 0 off, 1–63 hammer-on, 64–127 pull-off | M p.27 |
| 21 / 24 | Prefer open/low voicing / chord mode | see manual | M p.34 |
| 22 / 23 | Stroke speed (inter-string lag), fast / slow (emulated only) | ms | M p.35 |
| 25 | Pick buzz | | M p.35 |
| 42 | Round-robin mode | 0–31 off, 32–63 two, 64–95 three random, 96–127 four random | M p.35 |
| 43 | Legato / hammer / gliss sample volume | 0 maximum, 127 minimum | M p.20 |
| 44 / 45 / 46 | Configure Instrument Select assignments (not select an instrument) | instrument table on M p.40 | M p.40 |
| 48–51 | Mute mode and thresholds | | M p.53 |
| 52 | Remap the string/shape key-switch base note | string-6 note number; six consecutive notes follow | M p.54 |
| 53 | Forced string / chord shape | 0 off, 1–21 string 6, 22–43 string 5, 44–65 string 4, 66–87 string 3, 88–99 string 2, 100–127 string 1 | M p.25 |
| 54 | Auto Sustain | | M p.34 |
| 56 | Poly mode (single-note instruments) | | M p.35 |
| 57 / 58 | Auto-stroke resolution / direction mode | see section 1 | M p.36 |
| 62 | Release time | | M p.37 |
| 74, 28, 75–78, 80, 81 | Edit a Strum Key's settings | target key, target string, direction, string on/off, velocity rate, string type, extra strum noise | M p.44 |
| 82 | Extra strum-noise volume | | M p.43 |

- Other articulations:
  - Vibrato uses aftertouch, on single notes and dyads only. [M p.20]
  - Real-time legato slide: hold one Normal Key while playing the next. [M p.20]
  - Hammer-on, pull-off and trill are on single-note instruments. [M p.20]
  - Natural harmonics, FX1–FX4 and percussion are instruments. [M p.17–18]
- **Required ordering.** Articulation key switch, then string/shape key switch (or CC #53), then the sounding key. These must **not be simultaneous**. They need a slight timing offset. Key switches apply to the next note only. [M p.25–26]
  - CC #53 is the exception. It keeps the instrument on that string until a note exceeds the string's playable range. [M p.25]
- Auto Sustain keeps a note ringing after note-off until the next note-on or a Hold/Stop key. [M p.34]
- Articulation switching on a single MIDI channel is by design. [M p.34]

## 4. Strings, tuning, voicing

- **Strings.** Six strings, 1 (highest) to 6 (lowest). [M p.16, p.23, p.25, p.54]
- **Voicing.** Shape selection is documented. A voicing table is not available in extractable text.
  - Chord shapes 1–5 correspond to root strings 6 to 2, selected with the string key switches. [M p.25]
  - Repeating a shape key switch picks higher positions: once is lowest, twice is 12 frets up if available, three or more is the highest available. It is a 20-fret guitar, so no third octave. [M p.30–31]
  - "Prefer open/low" favours open or low voicings and disables legato slide. [M p.34]
  - Example arpeggio-key contents are given for Em7 (root on string 6): C3 string 1 = E, B2 = B, A2 = G, G2 = D, F2 = B, E2 = E. Aadd9 (root on string 5) leaves string 6 silent. [M p.23]
  - For an octave dyad, some strings are silent and one plays picking noise. [M p.23]
  - Per-chord shape diagrams are on M p.61–102 as images.
- **Tuning.** Verified against the Hummingbird manual [M] (full-text search of all 136 pages, re-run on 2026-11-22) plus two Kontakt pages from Native Instruments [NI1, NI2].
  - **Hummingbird itself: no tuning setting documented.** The manual states no string tuning, and no alternate tuning, drop-D, capo or per-string tuning control. A search for tuning/tune/capo/drop D/transpose/coarse/fine matched only Kontakt preload text [M p.12], the Stroke Speed "time lag between two strings" [M p.35], and the Chorus "detune" effect [M p.58]. The default tuning is not stated, so standard tuning is an undocumented assumption.
  - **Not tuning, but easily confused (all documented):**
    - *Pitch bend range* is per string (default 2 semitones, max 12, CC #28 selects the string and CC #30 sets 0–127 = 0–12 semitones; CC #29 sets all). It scales how far a bend moves a string. It does not retune the open string. [M p.53, "Pitch Bend Range"]
    - *Chord shape / user chord.* The User Chord editor (Options tab → "user chord") has Target User Chord, Rename, Always Open (forces a string to play open, 0 fret), Chord Shape, per-string Type ("how the string is played") and Root String. [M p.49–51] It defines fretting and string behavior, not string pitch. The Chord Shape/Type diagrams are images and were not read, so whether a user chord can shift a string's pitch is **unknown**.
    - *Shape/string key switches and CC #53* choose which string or shape plays. They are not tuning. [M p.25]
    - *Chord recognition and Instrument Select key switches* choose the chord type. They are not tuning. [M p.26, p.38]
  - **External Kontakt controls (not Hummingbird features).** Kontakt has an Instrument Header **Tune** knob (0 = A440) and a Master Editor **Master Tune** (frequency, default 440 Hz) that affect every instrument. [NI1, NI2] Both are global shifts of one instrument or the whole rack, so they cannot give per-string tuning. Whether Hummingbird follows these controls is **not documented** by Prominy, and was not tested. Do not rely on them for a guitar tuning.
  - **Unknown (not documented):** the actual tuning of the sample set; whether a sampled chord can be retuned; whether any hidden per-string tuning exists in the Options tab (the image-only pages were not read); any Kontakt 8 specifics.
  - Sources:
    - [M] <https://prominy.com/data/pdf/manual/en/Hummingbird/Hummingbird_user_manual.pdf>
    - [NI1] <https://support.native-instruments.com/support/solutions/articles/69000879720-how-to-master-tune-kontakt-to-a-specific-frequency>
    - [NI2] <https://support.native-instruments.com/support/solutions/articles/69000879773-my-kontakt-piano-instrument-sounds-out-of-tune> (Instrument Header Tune knob and Master Tune)
  - **Design implication (chosen plan).**
    - Represent single notes as absolute note names such as `C4`, then convert them to MIDI at the adapter using the manual's note-number convention.
    - Before mixing single notes with chords or strums, verify that the loaded Hummingbird patch produces matching pitches for the selected Normal Keys and its chord/strum samples. Until verified, keep mixed playback disabled.

## 5. What the plan's proposed profile can and cannot rest on

**Grounded (documented):**
- A key-switch/CC-driven articulation profile, using the tables above.
- Chord-type selection by key switch (e.g. major = E0, minor = D#0) or by chord-recognition notes. These are **defaults the user can change**. The profile should state them, and setup should verify them.
- Strum direction and extent through Strum Keys C2–C#3, or through direction CCs.
- Arpeggios by Arpeggio Keys E2–C3, one string per key.
- String and shape forcing by F7–A#7 (113–118) or CC #53.
- The ordering offset between selection and trigger.

**Not grounded or contradicted:**
- A mapping from "chord name + fret shape" to six ordered logical notes, then to MIDI pitches that spell the frets. The manual does not describe the instrument taking six pitches that spell a chord's frets.
  - Chord instruments are driven by one Normal Key or Strum Key, with chord type and shape chosen by key switch. The manual's own example inputs the chord by key switch and plays "G4" as the Normal Key. [M p.28]
  - The exact correspondence between note names, Normal Key values, and the loaded patch's sounding pitches must be verified in Kontakt.
  - The manual lets you select one of five chord shapes with String / Chord Shape Select keys [M p.25]. The User Chord editor lets you create a chord shape and choose its root string [M p.50]. The fret diagrams are images, so the exact mapping from an arbitrary `FretShape` to a built-in or user chord remains unverified.
- Exact voicing per chord. The tables are images.
- Auto stroke direction needs a running host transport with beat information. [M p.37] A headless host must supply that, or force direction with CC #58 or the Strum Key choice.

## 6. Uncertainties and gaps

- C3 = 60 is derived from the manual's table, not stated.
- Image-only pages (61–123, parts of 18–19) were not read.
- The text for the User Chord editor was read on p.49–51, but its diagrams were not extracted. An arbitrary `FretShape` mapping to a user chord remains unconfirmed.
- The manual lists Kontakt/Kontakt Player 5.8.1 or later and VST3 [M p.11]. It says nothing specific about Kontakt 8.
- No separate first-party MIDI implementation chart was found. The manual is the sole source.
- A third-party tutorial (Sleepfreaks) surfaced in search. It was **not** used as evidence.
