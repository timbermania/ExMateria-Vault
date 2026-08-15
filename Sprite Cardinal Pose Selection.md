# Sprite Cardinal Pose Selection

How FFT selects a unit's on-screen cardinal sprite pose from its 12-bit facing angle at `unit+0x70`: the PSX sprite renderer `FUN_8006bbfc` computes `cardinal_idx = (angle >> 10) & 3` (byte-boundary truncate, not center-snap) and passes it to `FUN_8017fddc`, which OR-merges it into a 4-bit field of a sprite-table entry that selects the pose downstream; the SHP data carries four distinct cardinal frames per state (front / side / back / mirrored-side), and the Godot reimplementation now matches the PSX truncate in `angle_12bit_to_facing` (unit-tested + chapel-trace verified) but its body renderer still runs a 2-pose (front/back + mirror) model that only reaches 2 of the 4 SHP frames, collapsing an 180° facing cascade to ~90°.

## Points

- **The PSX sprite renderer `FUN_8006bbfc` (0x8006bbfc) computes the cardinal pose index as `cardinal_idx = (angle >> 10) & 3` from the 12-bit facing at `unit+0x70` (block 0x8006bc1c–0x8006bc44: `lh` +0x70, sign-fixup dead for valid 0x000..0xFFF angles, `srl 10`, `andi 0xff`) — byte-boundary truncate, so bytes 0x0..0x3 → 0 (SOUTH), 0x4..0x7 → 1 (EAST), 0x8..0xB → 2 (WEST), 0xC..0xF → 3 (NORTH); Godot's `AnimationStateController.angle_12bit_to_facing` was rewritten from center-snap (±0x200 around byte anchors) to this PSX truncate with the anchor labels preserved, flipping mid-cascade poses at the PSX-truth boundaries (0xB/0x7) instead of the old snap boundaries (0xA/0x6).** — `[S·D·R] 3/3`
  - S: `0x8006bbfc` (renderer entry), `0x8006bc1c`–`0x8006bc44` (angle read + shift block) (`battle_disassembly.txt`; Ghidra's decomp mislabels the block `FUN_8006c3fc` due to an interleaved-blocks quirk)
  - D: chapel-trace live capture (`run_chapel_capture.py`, 90 s, PCSX 8082) + report (2026-06-27) — PC 97 (Agrias Rotate Unit Facing=2): Godot snaps 0x200 → SOUTH where the old center-snap gave EAST
  - R: `godot-learning/src/animation/AnimationStateController.gd` (`angle_12bit_to_facing`) — validated by `godot-learning/tests/UnitScenarioRotateTest.gd` (`_test_angle_12bit_to_facing_axis_values`, 12 axis assertions: 0x200 → SOUTH, 0xFFF → NORTH)
  - src: `research/working_documents/chapel_opcode_trace/HANDOFF_sprite_cardinal_mapping.md`
- **`FUN_8017fddc` consumes the cardinal_idx as its 5th argument and OR-merges it into a 4-bit field at offset 0x48 of a sprite-table entry (block 0x8017ff18–0x8017ff44); that field is what selects the cardinal sprite pose downstream.** — `[S] 1/3`
  - S: `FUN_8017fddc` (called at 0x8006bc44), `0x8017ff18`–`0x8017ff44` (OR-merge into the +0x48 4-bit field) (`battle_disassembly.txt`)
  - src: `research/working_documents/chapel_opcode_trace/HANDOFF_sprite_cardinal_mapping.md`
- **The polygon-visibility code splits the same 12-bit angle into uniform sectors (4 cardinal × 0x400 + 8 octant × 0x200 + 16 sub-octant × 0x100), and the decomp comment in DEPTH_MODE_RENDER_ORDER_GUIDE.md §"Stage 4" labels the cardinal bits NE/SE/SW/NW — screen-aligned ordinals, not compass N/S/E/W — so the S/E/W/N byte-anchor labels are internal FFT logical labels (on the uniform 22.5°/byte wheel S↔N is only 4 bytes = 90° and S↔W is 8 bytes = 180°).** — `[ ] 0/3`
  - src: `research/working_documents/chapel_opcode_trace/HANDOFF_sprite_cardinal_mapping.md`
- **The SHP data carries four distinct on-screen poses per state (type1_shp.json frames 1–4, parsed from TYPE1.SHP): frame 1 rectangle_x 0, no mirror (pose A, front/south), frame 2 rectangle_x 32, no mirror (pose B, side/east), frame 3 rectangle_x 64, no mirror (pose C, back/north), frame 4 rectangle_x 32 with mirror (pose B mirrored = west, reusing the east tile).** — `[D] 1/3`
  - D: `type1_shp.json` frames 1–4 (TYPE1.SHP dump parsed by `godot-learning/tools/parse_shp.py`; verified per doc 2026-06-27)
  - src: `research/working_documents/chapel_opcode_trace/HANDOFF_sprite_cardinal_mapping.md`
- **The Godot unit renderer runs a 2-pose (front/back + mirror) model instead of FFT's 4-cardinal frame selection: the resolution map pairs `back = body_slot + 1` with `facing_offset = 1 if use_back else 0`, and `_paint_body_variant` picks one of two SEQ anims (anim 2 → frame 2, anim 3 → frame 1) + mirror flag, so under all 4 cardinals it loads only 2 distinct SHP frames and never reaches frames 3/4 — the 180° Agrias chapel cascade (NORTH→WEST→EAST) collapses to ~90° because the NORTH→WEST step loads the same frame 1 and only toggles the mirror on the near-symmetric back sprite.** — `[D·R] 2/3`
  - D: headful side-by-side (PCSX 8082 + Godot ScenarioPlayer) user observation (2026-06-27): "the arrow turns 180 degrees (correct) but agrias only turns 90 degrees"
  - R: `godot-learning/src/units/Unit.gd` (`_paint_body_variant`), `godot-learning/src/animation/AnimationResolutionMap.gd` — no named test
  - src: `research/working_documents/chapel_opcode_trace/HANDOFF_sprite_cardinal_mapping.md`
- **The PSX sprite renderer also reads three sprite-state bytes at `unit+0x7c..+0x7e` in the same instruction block as the +0x70 angle (emitted as `pose_idx` / `pose_b` by the chapel probe, whose rows fire when any of them changes).** — `[D] 1/3`
  - D: `probe_chapel_opcodes.lua` / `run_chapel_capture.py` chapel capture (90 s, 194 state-change rows, 2026-06-27)
  - src: `research/working_documents/chapel_opcode_trace/HANDOFF_sprite_cardinal_mapping.md`

## Notes

(empty — user territory)

## Related

- [[Rotate Unit Interpolation]]
- [[Unit Anim Opcode]]
- [[Sprite Set Resolution]]
