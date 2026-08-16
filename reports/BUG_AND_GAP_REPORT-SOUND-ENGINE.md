# BUG & GAP REPORT — Sound Engine
**Project:** WARD NINE / HIGH-FIDELITY-GAME-PIPELINE  
**Date:** 2026-08-16  
**Scope:** Audio subsystem only (live TanStack build + pack baseline)  
**IP:** SUNDER / WARD NINE only — no Toho / Ifukube recordings

---

## Executive summary

The engine pack ships a **beeper**, not an HD sound engine. The Tokyo design doc (Phase Audio + Audio-2) specifies a sacred layered roar, adaptive stems, spatialization, and per-mode foley. Live WARD NINE advanced slightly beyond the pack (one synthesized stereo roar WAV + rain bed + mute), but still fails the design bar.

Capability note from the pack is correct: Web Audio already supports the full model (`GainNode` buses, `PannerNode`, `ConvolverNode`, multi-stem `BufferSource`s). What's missing is the **graph + the banks**, not a new platform.

---

## What was actually in the engine pack

| Asset | Reality |
|---|---|
| `lab/app/src/game/audio.ts` | ~80 lines. `AudioContext` + **master** + **sfx bus** (gain 0.35). Oscillator one-shots: `sfxFire`, `sfxPop`, `sfxHit`, `sfxWin`, `sfxStart`. Mute via `setTargetAtTime`. |
| Milkshake `world.audio` | Handle on the runtime. Not a mixer. |
| CAPABILITY-2026-08-15 | *"Audio too primitive" is misattributed. Adaptive audio ≈ 150 lines **plus the stems**.* |
| Roar bank / orchestra stems / WAV / OGG | **None in the zip.** |

Sacred-roar architecture lives in `PROCEDURAL_TOKYO_SYSTEM_DESIGN(21).md` (Phase Audio), not in the runtime.

---

## Bug report — live / shipped audio

| ID | Sev | Bug | Notes |
|---|---|---|---|
| **AUD-01** | P0 | **Wrong engine topology.** Live mixer is often a single `GainNode`. Pack already had **master + sfx bus**. Bus structure was dropped or never restored. | Restore pack bus layout first. |
| **AUD-02** | P0 | **Roar is a mono event, not a system.** Spec: shriek / bellow / rumble / city IR / distance. Live: one 3.4s stereo WAV + a 1.35s crop. No intensity, no distance, no occlusion. | Replace with layered instrument. |
| **AUD-03** | P0 | **Roar over-fires.** Full sacred roar on every mode enter. Spec: *never overuse — earned only.* | Cooldown + tiered triggers. |
| **AUD-04** | P1 | **Unlock / decode race.** First gesture can start SFX before `decodeAudioData` finishes → fallback beep, then real roar stacks late. | Await decode before sacred trigger. |
| **AUD-05** | P1 | **No roar ducking.** Rain + beeps sit on top of the roar. Spec: sidechain everything in 50–80 ms. | Sidechain compressor into music/ambience/sfx. |
| **AUD-06** | P1 | **Point-source, not 100 m.** No `PannerNode`, no source width, no canyon IR. | Spatial + convolution path. |
| **AUD-07** | P1 | **Shake not locked to bellow.** Camera shake is a timer, not keyed to the roar's low-end envelope. | Drive shake from audio envelope. |
| **AUD-08** | P1 | **No haptics.** Spec: rumble profiles on intensity + proximity. Zero `navigator.vibrate` / gamepad pulse. | Map intensity tiers → pulse patterns. |
| **AUD-09** | P2 | **Mute is a hard cut.** Pack used `setTargetAtTime`. Hard `gain.value` writes click. | Always ramp. |
| **AUD-10** | P2 | **No voice limit / steal.** Roar + smash + breath + rain can stack with no cap. | 8–16 voice pool; roar never stolen. |
| **AUD-11** | P2 | **WAV only, uncompressed.** Fine for two files. Will blow budget when stems land. | Prefer Opus/AAC + decode cache. |
| **AUD-12** | P2 | **Synth roar DNA is off.** Current bake is rumble-forward. Missing metallic glove-on-bass **shriek → earth bellow → long tail** (C–D center, octave drop). | Re-author layers from design DNA. |
| **AUD-13** | P3 | **No persistence.** Mute doesn't save. No `latencyHint: "interactive"` in some live builds (pack had it). | localStorage + interactive context. |

---

## Gap list — required for a real sound engine

### A. Mixer (~150 lines of Web Audio graph)

- Buses: `master` → `roar` / `sfx` / `foley` / `ambience` / `music` / `ui`
- Roar bus has **priority + sidechain** into music / ambience / sfx
- Compressor / limiter on master
- Mute / duck as ramps, not hard zeros
- Voice pool (8–16) with steal rules: roar never stolen

### B. Roar bank (the real missing pack asset)

| Layer | Role | Status |
|---|---|---|
| Core shriek | High metallic attack | Missing |
| Body bellow | Mid resonance | Weak (baked into one file) |
| Rumble bed | Sub / felt | Weak |
| City resonance | Convolution vs Ward Nine | Missing |
| Distance set | Close / mid / far / through-buildings | Missing |
| Intensity | 0.2 warn / 0.5 threat / 1.0 sacred | Missing |
| Variants | Form 1 strained → Form 4 catastrophic | Missing |
| Triggers | `{ type, t, intensity, position, variant }` | Missing |

**IP rule:** no Toho / Ifukube recordings. Original layers only.

### C. Adaptive score (Phase Audio-2)

- Stems: pedal, ostinato / footstep, melody, brass, percussion, tension, choir
- States: calm / SUNDER-near / roar / aftermath
- Duck under roar 50–80 ms
- Pack truth: *stems are the content; the graph is small.* Neither is shipped.

### D. Per-mode foley (design spec, not in pack)

| Mode | Specified | Live baseline |
|---|---|---|
| LEDGE | Jump / land weight, ladder, rescue chime, hazard telegraph, **distant roar bed** | Thin beeps |
| CROSS | Engine, tire, metal crunch (score-linked), tank warn, train horn, **topple sequence**, move rumble | Smash beep |
| MAZE | Stomp-by-surface, collect, chase stingers, breath charge → roar, powered vs fading | Beep |

### E. Presence, not just one-shots

- Off-screen rumble before SUNDER is visible
- Footstep / stomp as a loop scaled by speed
- BlockPackage `audio_events` (`roar_trigger`, `rumble_bed`, `music_stem`)
- Predictive preload — only after banks exist

### F. Sync + feel

- Shake + haptic keyed to bellow transient
- Lights flicker on sacred roar
- Chromatic / desat pulse on full roar
- Cooldown so sacred roar cannot retrigger for N seconds

---

## Suggested update order

1. **Restore pack mixer** (buses, ramps, voice limit, `latencyHint: "interactive"`).
2. **Roar as a layered instrument** — 3–5 stems, intensity + distance, sidechain, cooldown. Replace the single WAV.
3. **Stop firing sacred roar on menu → mode.** Distant bed on LEDGE; full roar only on sweep / topple / first sighting.
4. **Mode foley table** (smash tiers, breath charge, stomp, rescue).
5. **Two music stems** (pedal + presence). Full orchestra after the roar is actually scary.

---

## Non-goals / constraints

- Do **not** ship copyrighted Godzilla / Toho / Ifukube audio.
- Do **not** treat another oscillator bank as "done."
- Do **not** over-trigger the sacred roar; overuse kills the reaction the design demands.

---

*End of sound-engine bug & gap report.*
