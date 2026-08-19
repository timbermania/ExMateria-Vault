# Web-psx Cross-Validation

An external cross-validation pass over this vault by **web-psx**, a PlayStation
emulator written as a programmable PSX debugger and FFT reverse-engineering
harness (JIT-to-JavaScript CPU/GTE, WebGPU GPU, WebAudio SPU; a Lean 4 sibling
emulator, lean-psx, is the semantics reference). Our evidence standard is an
instrumented emulator running the retail game off the real disc image: a claim
is *graded* by re-deriving it two ways — statically out of `BATTLE.BIN` /
`SCUS_942.21` on the disc, and dynamically by running the game with a shadow
reimplementation *beside* it and byte-diffing every write (`tools/fxsim.ts` for
`EFFECT/*.BIN`, `tools/eventsim.ts` for the event VM), plus GPU packet capture
at `DrawOTag` and a hardware-verified CPU/GTE (55 × 1000 single-step vectors,
GTE captures, JaCzekanski's hardware suite). Every number below is reproducible
from a disc image and a BIOS with no hand-driving, so these claims grade to this
vault's **D** as well as **S**. This note carries (a) the claims of yours we
independently confirmed, (b) a summary of the corrections landed as
`⚠ SUPERSEDED` sub-bullets on the specific points, (c) a proposal for a mutual
camera cross-validation, and (d) tooling notes for raising `[S] 1/3` to `[D]`.
Contributed 2026-08-19; contact is via the pull request that carries this note.

## Points

- **Every event-VM builder address in this vault's presentation notes matches web-psx's independently-read dispatch ladder exactly, including `{73}`'s and `{38}`'s bytecode pre-patchers.** — `[S] 1/3`
  - S: `src/events/dispatch.ts` reads the 145-rung ladder at `80143d0ch` out of `BATTLE.BIN`'s own image rather than transcribing it; `npx vite-node tools/eventsim.ts --table` prints `19 → 80146110 (BATTLE.BIN+df110)`, `1a → 801466b4 (+df6b4)`, `1d → 8013db9c (+d6b9c)`, `2a → 8013e904 (+d7904)`, `3e → 801467dc (+df7dc)`, `6a → 80149a54 (+e2a54)`, `6b → 801499ac (+e29ac)`, `76 → 8013bd94 (+d4d94)`, `47 → case body 801456ac`, `73 → case body 80144c10 calling 801474a4`, `38 → case body 80144be8 calling 80147780` (2026-08-19)
  - src: external contribution — web-psx `docs/event-seam.md` [event.hle.dispatch]
- **The 40-halfword direction block at `BATTLE.BIN+0x10dc` is byte-exact as this vault reads it, and its sub-tables A and B agree 16 of 16 with web-psx's independently fitted octant formula — which converts an empirical 88.8% fit over 70,466 actors into a ROM-table identity.** — `[S] 1/3`
  - S: dumped off the US retail disc via `src/cdrom/iso.ts`: `+0x10dc` (F, SEQ mirror by cardinal) = `0,0,2,2`; `+0x10e4` (E, SEQ anim offset by cardinal) = `0,1,1,0`; `+0x10ec` (B, idle mirror by octant) = `0,0,0,0,0,0,0,0,0,2,2,2,2,2,2,0`; `+0x110c` (A, frame base by octant) = `1,2,2,3,3,4,4,5,5,4,4,3,3,2,2,1` (2026-08-19)
  - src: external contribution — web-psx `docs/scene-viewport.md` [viewport.sprite], `tools/combatdata.ts` `animDirection`
- **The Charging Pose Set Table's sampled sets are byte-correct, and sprite categories 5–7 hardcoding animation `0x2C` reproduces independently.** — `[S] 1/3`
  - S: `BATTLE.BIN+0x2cde8` dumped: set 0 = `2a/22`, set 1 = `2a/2b`, set 7 = `00/22`, set 10 = `2a/22` — all four as claimed; the categories 5–7 short-circuit is web-psx's `BATTLE.BIN+1bc10` reading (Lucavi / the two Altimas) (2026-08-19)
  - src: external contribution — web-psx `docs/combat-tables.md` [combat.animation]
- **The Weapon Animation Table's *values* are byte-for-byte correct and `unit+0x13b` is indeed the index; only the prose ordering of the weapon types is wrong (see [[Weapon Animation System]]).** — `[S] 1/3`
  - S: `BATTLE.BIN+0x2d364` dumped, 19 rows × 3 (2026-08-19); `QueueSpriteAnim(1,…) → WEP1` / `(2,…) → EFF1` matches web-psx's `QUEUE_SPRITE_ANIM = 0xfff2` in `src/sprite/seq.ts`
  - src: external contribution — web-psx `docs/combat-tables.md` [combat.animation]
- **The particle-curve resolution and the emitter stride were arrived at independently and agree exactly: `factor = u8[curve_table + N*0xA0 + frame + 4]`, and an emitter is 196 bytes at `data + 0x14 + index*0xC4`.** — `[S·D] 2/3`
  - S: web-psx `src/effects/records.ts`: `CURVE_HEADER = 4`, `CURVE_STRIDE = 160`, `EMITTER_HEADER = 0x14`, `EMITTER_STRIDE = 196`, effect records at `801bf02c` stride 248
  - D: three effect shapes transcribed from the overlays' own MIPS and run beside the console's copy, byte-diffing every write into the work block, the effect record, the ordering table and the scratchpad — 52 files, 100,000+ brackets, **0 disagree** (web-psx `docs/effect-format.md` [effect.hle.score]; cross-referenced 2026-08-19)
  - src: external contribution — web-psx `docs/effect-format.md` [effect.hle.score]
- **The finding that emitter fields around `0x4C` are callback parameters repurposed as CLUT/blend-mode bits is independently corroborated, as is `get_camera_position` as the emitter anchor ladder's camera mode.** — `[S·D] 2/3`
  - S: web-psx's transcription of the ring and column shapes reads `+0x4c`/`+0x4e` for exactly this, and reads the tpage/semi-transparency word out of `+0x24` on the column shape — the same non-obvious repurposing this vault found in E317's CB91/CB92
  - D: the anchor ladder scored over 26 files and 54,495 brackets with **0 disagreements**; the CAMERA mode resolves to the map centre from the width/height bytes at `800e4e9c`/`800e4ea0` (web-psx `docs/effect-format.md` [effect.hle.column.score]; cross-referenced 2026-08-19)
  - src: external contribution — web-psx `docs/effect-format.md` [effect.hle.column.where]
- **`{6B}`/`{6A}` background sound checks end-to-end against our own decompile of `BATTLE.BIN+e29ac` and `+e2a54`, including that byte 1 is StartVol and not the wiki's "Echo".** — `[S] 1/3`
  - S: operands `[Sound, StartVol, Volume, Stacking, Time]`, task id `0x35`, `handle = 0x10000 | Sound` — bank id 1 is `SOUND/ENV.SED`, which is exactly the namespace web-psx's `src/jukebox/sed.ts` predicts from the bank header's `+0ah`; the ramp worker yields once per tick as `a1 = trunc(k·delta/Time) + Start`, `abs()`, `if 0 then 1`, then one final **exact** write of `Volume`. The opcode lands on three sound-driver entrypoints web-psx already taps (`+3190`, `+2d60`, `+2da8`) (2026-08-19)
  - src: external contribution — web-psx `docs/event-seam.md`, `docs/audio-seam.md` [audio.door.doors]
- **This vault corrected *us*: `SCUS_942.21+336c` (your `FUN_80012b6c`) is set-volume-by-handle, not the conditional stop our own documentation called it — and it is our highest-frequency game-side sound entrypoint.** — `[S·D] 2/3`
  - S: decompiled `SCUS_942.21+336c`: for each of 8 slots (stride `0x160`) whose stored id at `ch+8h` matches, it writes `vol<<8` to `ch+94h` and sets the dirty word `ch+2h = 0x100`; **only** when `vol == 0` does it take the clear-and-key-off path via `sub_80012ab0` (2026-08-19)
  - D: 997 calls in a battle window and 1,677 over a recorded session, previously labelled as stops in web-psx `docs/audio-doorlog.md` [doorlog.vocabulary]
  - src: external contribution — web-psx `docs/audio-seam.md` [audio.door.doors]
- **The `{63}` ease closed form in [[Scenario Camera Opcodes]] is algebraically identical to web-psx's independent derivation from the same branch web — two independent readings of the same curve.** — `[S] 1/3`
  - S: our form is `f = (k·u² + (16−k)·u)/32` for the first half with `u = 2t/T`, which is exactly your `prog(t) = (16−I)/16·t + I/16·easeQuad(t)` rewritten; the field split `curve = m & 3`, `group = (m>>2) & 3`, `k = (m>>4) & 15` is confirmed (lean-psx `docs/kb/BATTLE.BIN/000ff054`, `CAMERA_EASE_MODE @ 80166054h`)
  - src: external contribution — lean-psx `docs/kb/BATTLE.BIN/000ff054`
- **Your Map Tint supersession is independently confirmed by VRAM measurement, and your damage-number fade mechanism is corroborated by our own packet capture.** — `[S·D] 2/3`
  - S: web-psx `docs/scene-viewport.md` [viewport.dim.clut]
  - D: through a whole cast, VRAM movement is rows 480–484, 487 and their staging copies **and nothing else** — the ×8 terrain-vertex applier you retracted by Ghidra xref does not run; separately, the damage number appears in our `DrawOTag` capture as `2c page 001f clut 79c7`, 72 quads, matching your `tpage 0x001F` reading (web-psx `docs/effect-format.md` [effect.where.primitives]; cross-referenced 2026-08-19)
  - src: external contribution — web-psx `docs/scene-viewport.md` [viewport.dim.clut]
- **Corrections landed as `⚠ SUPERSEDED` sub-bullets, sprite/ability domain: the height table's non-standard entries, the weapon-type index ordering, the Ability Animation Table's row count, and the headline camera-relative claim.** — `[S] 1/3`
  - S: [[Unit Sprite Height Table]] (the shipped US table has 4 distinct heights, not the 5 claimed), [[Weapon Animation System]] (index order at `+0x2d364`), [[Ability Animation Table]] (454 rows, not 512; 70 abilities with `effect_anim_id == 0`, not 96; RAM base `0x80093c10`, not `0x8003CE10`), [[Sprite Cardinal Pose Selection]] (`FUN_8006bbfc` carries no camera term), [[Damage Number Popup System]] (the layer table is 24 × 16 bytes) — each with the dump inline (2026-08-19)
  - src: external contribution — web-psx `docs/combat-tables.md` [combat.animation]
- **Corrections landed as `⚠ SUPERSEDED` sub-bullets, effect/sound domain: the particle tpage formula, the effect file buffer's 4-byte prefix and load base, the `.SED` stream-extent rule, and two SMD/FEDS opcode rows.** — `[S·D] 2/3`
  - S: [[Particle Emitter Format]], [[Effect File Buffer]], [[FEDS Sound Definition Format]]
  - D: 4,919 GPU packets from a live cast contradict `tpage = (animation_word & 0xE0) | 0x08`; 183,176 words of DMA provenance put an effect file's byte 0 at `801c2500` with no prefix; a census of 2,388 streams in 514 banks shows the next-stream-start rule truncates 94 of them (web-psx `docs/effect-format.md` [effect.where.primitives], `docs/modules.md`, `docs/audio-seam.md` [audio.smd.sed]; cross-referenced 2026-08-19)
  - src: external contribution — web-psx `docs/effect-format.md` [effect.where.primitives]
- **Corrections landed as `⚠ SUPERSEDED` sub-bullets, event/scenario domain: `{1E}`'s handler attribution, the `{7E}`/`{7F}` CONTESTED pair, the block skip's one-byte tail, the `{63}` low-nibble semantics, and the ENTD facing labels.** — `[S] 1/3`
  - S: [[Event Opcode Catalog]] (`0x8013db9c` is `{1D}`'s builder; `{1E}` has no rung), [[Wait Value Opcode]] (`7e` and `7f` have distinct case bodies at `80145afc` and `80145b14` — no delay-slot story needed), [[Block Execution]] (the forward scan does not end the instruction), [[Scenario Camera Opcodes]] (`curve == 1` is decelerate, not ease-in), [[ENTD Unit Deployment Table]] (the cardinal labels contradict this vault's own `+0x70` wheel) — each with the falsifier inline (2026-08-19)
  - src: external contribution — web-psx `docs/event-seam.md` [event.hle.dispatch]

## Reproducing A Continuous Camera Pose Track

[[Scenario Beat Capture]] is a good protocol and it concedes its own limit:
*"camera framing at the same PC still differs (scenario director)"*. That is not
a Godot bug to hunt so much as a **missing measurement** — a beat is a
freeze-frame, and a cinematic camera is a *track*. The fix costs one Lua script
and no new tooling on your side, and it produces the one artefact that makes the
two engines comparable frame for frame.

**The capture.** You already run a non-pausing exec BP on the per-vsync camera
ticker `0x801439c0` (that is how the `{63}` ease fixture was measured). Keep it,
and instead of sampling one field, dump the whole live pose per tick and append
a row. Park on the chapel savestate, let the whole cinematic run untouched, and
you have a pose track for scenario 1 with no beat parking at all. Two readings
of the same pose are available and they should be dumped together, because a
disagreement between them is itself informative:

- **The scratch struct** this vault already names: `+0x68` X, `+0x6c` Y, `+0x70`
  Z, `+0x74` pitch, `+0x78` yaw, `+0x7c` roll, `+0x80` zoom — copied each vblank
  into the GTE mirror at `0x8016e3f0…e408` with no curve math.
- **The event variables the `{19}` task actually reads and writes**, which is the
  authored pose in the script's own units. The task takes its ids from
  `CAMERA_VAR_IDS` at `80165ED4h` against the variable array at `*(0x80165F9C)`:
  ids `0x1A`/`0x1B`/`0x1C` are X/Z/Y **stored `<<10`**, and `0x1D`/`0x1E`/`0x1F`/
  `0x20` are Angle/MapRot/CamRot/Zoom raw. `0x1E` is pinned as Map Rotation by
  the task's own one-shot unwrap, which reads variable `1Eh` by name
  (`801461c8h`). Seven words a tick.

**Our numbers for the same segment, published as the cross-check.** web-psx
captured scenario 1 / MAP062 / ENTD 256 (aux opens event 2) and read the camera
off the producer's own wire bytes — **1,911 records over frames 3819–5999**,
about 36 seconds, covering the opening sweep, the whole *"God, please forgive us
sinful children of Ivalice"* narration, and the cut past it. `eye` is where the
camera stands, `centre` what it looks at, `yaw` the game's own 4096-to-a-turn
heading, `zoom` screen pixels per world unit:

| frame | eye | centre | yaw | zoom |
| --- | --- | --- | --- | --- |
| 3819 | (1655, 1217, −1657) | (392, 322, −392) | 3584 | 0.999 |
| 3943 | (604, **2201**, 119) | (−341, 440, 121) | 3074 | **1.888** |
| 4048 | (974, 2035, 107) | (−223, 434, 120) | 3079 | 1.664 |
| 4151 | (1371, 1804, 44) | (−76, 427, 108) | 3101 | 1.400 |
| 4252 | (1804, 1478, −421) | (150, 439, −4) | 3233 | 1.028 |
| 4355 | (1707, 1361, −1146) | (227, 420, −188) | 3446 | 0.954 |
| **4457** | **(1444, 1271, −1531)** | **(182, 377, −267)** | **3584** | 0.999 |
| … | *(unchanged for 16 s)* | | | |
| 5484 | (1248, 1271, −1711) | (139, 377, −309) | 3659 | 0.999 |
| 5587 | (−1309, 1271, −1911) | (−427, 377, −356) | 4432 | 0.999 |

Read as motion, and these are the four claims a Lua pose track either reproduces
or does not:

1. **It opens already moving.** By frame 3943 the camera is at a close framing —
   zoom 1.888, nearly twice the resting scale — high above the chapel at eye
   Y = 2,201, yaw 3074.
2. **It descends and pulls out** over about 300 frames (5 s): eye Y
   2201 → 1478 → 1271 while zoom decays 1.888 → 1.028 → 0.999. The descent is
   **930 units**.
3. **It turns 45° while it falls**: yaw 3074 → 3584, i.e. 510 of 4096.
4. **It arrives at frame 4503 and holds for 945 more records** — 16 seconds at
   eye (1444, 1271, −1531), yaw 3584, zoom 0.999. That is the narration pose:
   the whole of *"God, please forgive us"* is spoken at one camera. **Then it
   cuts** at 5587 to yaw 4432 (336 mod 4096, a 74° turn) and 2,750 units across,
   settling again at 5599 and holding 198 more records.

Both arrivals are detected by a settle rule (still to within half a unit and a
thousandth of a turn, held a second), so frames **4503** and **5599** are
measurements and not eyeballs.

**How to compare.** Our poses are the *consumer's* camera (eye/centre in world
units), not the raw event variables, so do not expect the numbers to match
component for component; and our frame indices are our corpus run's, so align on
the first camera move rather than on an absolute index. What should match
exactly, because they are the game's own quantities and not anybody's
projection, is the **yaw in 4096ths**, the **zoom ratio**, and the **shape of
the track** — the descent's magnitude, the 510-unit turn, the frame the track
settles at, and how long it holds. If your Lua trace reproduces those, two
independent toolchains have validated each other on a real cinematic, which is
worth more to both of us than either capture alone. If it does not, the
disagreement is interesting to both sides and we will chase it from our end.

**The decoder for what you capture.** web-psx and lean-psx hold the `{19}` task
in full (lean-psx `docs/kb/BATTLE.BIN/000df110`), and five of its behaviours are
things a pose track will otherwise look like noise without:

- **`0x2710` (10000) is the per-field "leave alone" sentinel on *every*
  component**, tested in the animation loop (`801462FCh`) and again twice in the
  snap-to-target epilogue (`801465CCh`, `8014661Ch`).
- **A one-shot `+4096` map-rotation unwrap.** If `801696F8h` is non-zero — it is
  set to 1 once, by the event engine's startup at `80143F94h` — the first
  `Camera` of a scene adds one whole turn to the stored yaw when that takes the
  interpolation the short way round. Only `+4096` is ever considered, never
  `−4096`. A pose track that starts with an unexplained 4096-unit jump is this.
- **Two dead zones.** A move of under 1536 in 1/256 units (**6 script units**) on
  X/Z/Y, or under 96 in 1/64 units (**1.5 units**) on the rotations and zoom, is
  not animated at all — it only lands in the epilogue's snap
  (`80146350h`, `80146374h`).
- **A genuine rounding bug at `0x80146538`**, and we think it is your ±1/1024
  residual. The round-to-nearest there masks with `0FFh` against a **6-bit**
  fixed point, so bits 6–7 of the integer part leak into the test: for a positive
  value `(v & 0FFh) < 33` only passes when `(v >> 6) & 3 == 0`, and three times
  out of four it adds a whole unit. It perturbs Angle, MapRot, CamRot and Zoom by
  ±1 unit during a move. Your measured fixture moves MapRot 1536 → 2560 — a
  1024-unit move — so ±1 unit *is* ±1/1024 of that move. The same rounding done
  correctly appears at `8013E4C0h` in camera fusion, where the mask is `0FFFh`
  against a 12-bit fixed point. Reproduce it literally.
- **The epilogue writes the exact parameters regardless**, so an implementation
  only has to get intermediate frames approximately right but must land exactly.

One clock question worth pinning while you are in there: we read `Time` as
**task ticks** — one pass of the 16-slot fiber scheduler at `8014CA80h`, which
during an event is one game frame, two video frames, 1/30 s — rather than
vblanks. Your `{63}` fixture moved 1536 → 2560 over "exactly 64 frames", which
agrees if those samples were game frames and disagrees if they were vblanks. The
pose track settles it in one run.

## Tooling Notes

Offered because several `[S] 1/3` points here are one cheap run away from `[D]`,
and because two of the corrections above are the kind a shadow diff finds
automatically.

- **Bound a capture by the machine's clock, not by wall time.** Capture
  identifiers in this vault are shaped like "90 s chapel capture" and "30 s".
  A wall-clock window is not reproducible across hosts and cannot be diffed
  against a later run; an emulated frame count or an instruction count can. Our
  equivalents name a frame range (`frames 3819–5999`) or a packet count, and a
  re-run reproduces them exactly.
- **Grade `[R]` claims with a diff shadow, not with eyeballs.** The pattern that
  found every effect-system error we know of: run the reimplementation *beside*
  the emulated game, in the same process, and diff **every byte** it writes into
  the structures the game writes — the work block, the effect record, the
  ordering table, the scratchpad. `tools/fxsim.ts` does this for `EFFECT/*.BIN`
  (3 shapes, 52 files, 100,000+ brackets, 0 disagree) and `tools/eventsim.ts`
  does it for the event VM, diffing every dispatched instruction and moving on
  past presentation opcodes it deliberately does not implement. A Godot port
  cannot run inside the emulator, but the same shape works across a socket: emit
  the reimplementation's per-frame state and diff it against the emulator's, and
  an `[R]` badge starts meaning something a test can fail on.
- **Prefer a static walk that has to close.** The cheapest falsifier we have for
  a table read out of the binary is one where a wrong answer cannot land: a
  script slot's first word is its text-section offset, so a width-table walk must
  land on it exactly — 304 of 304 do, which validates all 176 widths at once and
  showed that the 80 opcodes FFTPatcher leaves undefined are 1-byte instructions
  rather than undefined. Table offsets can be checked the same way: our
  `SCUS_942.21` table set is validated by *tiling* — each table's offset plus its
  length lands on the next one's — and it is that property which shows the
  Ability Animation Table stops at 454 rows.
- **Take savestates at instruction boundaries.** A state taken at a block
  boundary composes with an input recording, so a moment can be resumed and
  re-measured rather than re-navigated; our replay is deterministic to a
  checkpoint hash, which is what lets a measurement be *re-run* six months later
  rather than re-argued.
- **Read the dispatch, don't transcribe it.** Both of the event-VM attribution
  errors corrected above (`{1E}`, `{7E}`/`{7F}`) fall out of walking the
  interpreter's own comparison chain in the binary and classifying each case body
  by what it calls. It is 145 rungs; a scan that only follows `$v0` loses one of
  them (`3bh`, which the rung leaves in `$s0`), and that is the whole difficulty.

## Notes

(empty — user territory)

## Related

- [[Ability Animation Table]]
- [[Block Execution]]
- [[ENTD Unit Deployment Table]]
- [[Effect File Buffer]]
- [[Event Opcode Catalog]]
- [[FEDS Sound Definition Format]]
- [[Particle Emitter Format]]
- [[Scenario Beat Capture]]
- [[Scenario Camera Opcodes]]
- [[Sprite Cardinal Pose Selection]]
- [[Unit Sprite Height Table]]
- [[Wait Value Opcode]]
- [[Weapon Animation System]]
