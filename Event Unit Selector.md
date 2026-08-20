# Event Unit Selector

The event unit-selector operand (`V = Units | Multi<<8`) shared by `{11}` Unit Anim, `{2D}` Rotate Unit, `{53}`, `{32}` Color Unit, and March is resolved by `resolve_event_unit_handle` into a mode that drives a 21-slot membership loop in the per-frame color command processor; `V == 0` is PSX's broadcast-to-all-existing-units path (mode 1), not single unit 0 — which Godot's `ScenarioDecode.unit_set_mode` now maps to `UnitSetMode.ALL`.

## Points

- **`resolve_event_unit_handle @ 0x80147928` maps selector `V` (`Units | Multi<<8`) to a mode: `0 < V < 0x100` → mode 0 (single unit id V, loop breaks after one), `V ≥ 0x100` → mode `V − 0xFE` (team-set broadcast), `V == 0` → mode 1 (broadcast — all units).** — `[S·D·R] 3/3`
  - S: `resolve_event_unit_handle @ 0x80147928` (`battle_decompilation.c`)
  - D: scenario-8 read-only RAM/VRAM capture (2026-07-10): 81 view-3 unit-palette entries consistent with an all-present-units broadcast
  - R: `godot-learning/src/scenarios/ScenarioDecode.gd` `unit_set_mode`, validated by `godot-learning/tests/ScenarioDecodeTest.gd`
  - src: `research/working_documents/COLOR_TINT_LUMA_MODE_SEPIA.md`
- **`FUN_801479ac @ 0x801479AC` is the per-candidate membership test for the 21-slot selector loop: mode 1 = any existing unit (`unit_sprite_object_exists` only, no team/alive filter), 2 = team A/player (`(unit+5) & 0x30 == 0`), 3 = team A + alive, 4 = team B/enemy (`(unit+5) & 0x30 != 0`), 5 = team B + alive.** — `[S] 1/3`
  - S: `FUN_801479ac @ 0x801479AC` (`battle_decompilation.c`)
  - src: `research/working_documents/COLOR_TINT_LUMA_MODE_SEPIA.md`
- **An event `{32}` reaches many units via the selector membership loop in `event_color_command_processor @ 0x801495E0` — read selector, resolve, loop up to 21 candidate slots, one `color_unit_fanout` call per member, break if mode==0 — which is distinct from `color_unit_fanout`'s own all-16 branch (param ≥ 0x10) that fires only for effect/charge callers; the event path always passes a single resolved handle per call.** — `[S] 1/3`
  - S: `event_color_command_processor @ 0x801495E0`, `color_unit_fanout @ 0x800933C4` (`battle_decompilation.c`)
  - src: `research/working_documents/COLOR_TINT_LUMA_MODE_SEPIA.md`
- **scn8 PC29's `{32}` has `Units=0, Multi=0` → `V=0` → mode 1, so it tints ALL present units (PSX cannot single-target unit 0 — id 0 always broadcasts); Godot's `ScenarioDecode.unit_set_mode(0, 0)` now returns `UnitSetMode.ALL` (was `SINGLE`), and because this is the shared `{11}`/`{2D}`/`{53}`/`{32}`/March selector, `(0,0)` now broadcasts for all of them with zero new failures in a full scenario regression sweep.** — `[S·D·R] 3/3`
  - S: `resolve_event_unit_handle @ 0x80147928` V==0→mode-1 fall-through + PC29 operands (`battle_decompilation.c`, `godot-learning/assets/scenarios/chunks/scenario_008_chunk.json`)
  - D: scenario-8 read-only RAM/VRAM capture (2026-07-10): 81 view-3 entries (many units) confirm the all-present-units broadcast
  - R: `godot-learning/src/scenarios/ScenarioDecode.gd` `unit_set_mode`, validated by `godot-learning/tests/ScenarioDecodeTest.gd` `_test_unit_set_mode` (`(0,0)→ALL`)
  - src: `research/working_documents/COLOR_TINT_LUMA_MODE_SEPIA.md`

- **`V == 0` broadcasting to every existing unit is confirmed exactly, and it is the one claim in this vault that needed no correction at all.** — `[S] 1/3`
  - S: the routine is `ClassifyUnitSpec` at `BATTLE.BIN+e0928`, and **11 rungs share it** — `0x11`, `0x2C`, `0x2D`, `0x32`, `0x53`, `0x69`, `0x6C`, `0x6D`, `0x80`, `0x81`, `0x83` — while `{47}` is *not* one of them. The `V == 0` arm is 2 instructions: `0x80147944` branches to `0x80147988`, which sets membership mode 1 and returns, and mode 1's per-slot predicate is `FindActor(i) != 0` over `i = 0..0x14`, i.e. *exists*. For `0 < V < 0x100` the resolved id is written back into the spec word **in place**, and `0x7D0` coming back is the routine's only false. A model reading `Units=0, Multi=0` as "unit 0" would aim 96 of the disc's 6,505 `0x11`s, 89 of its 620 `0x32`s and 27 of its 808 `0x2D`s at one unit instead of at all of them (web-psx `docs/event-seam.md` [event.hle.units]) (2026-08-19)
  - src: external contribution — web-psx `docs/event-seam.md` [event.hle.units] (see [[Web-psx Cross-Validation]])

## Notes

(empty — user territory)

## Related

- [[Color Tint Luma Modes]]
- [[Event Opcode Catalog]]
- [[Unit Anim Opcode]]
- [[Rotate Unit Interpolation]]
