# Particle Emitter Format

The on-disk 196-byte (0xC4) ParticleEmitter record embedded in each E###.BIN's Particle System section (at `effect_data_ptr + 0x14 + index*0xC4`). Emitters are pure static configuration — a 16-byte core control block, per-parameter nibble curve indices, and int16 min/max start/end range pairs for every physics parameter (position, spread, velocity angle/spread, inertia, weight, radial velocity, acceleration, drag, lifetime, target offset, spawn, homing). Callback parameters and child-emitter slots round out the record; blend mode is NOT here (it is baked into the animation sequence word).

## Points

- **A ParticleEmitter is a 196-byte (0xC4) static record at `effect_data_ptr + 0x14 + emitter_index * 0xC4`, whose core control bytes 0x00–0x0F are: byte_00 unused padding, anim_index 0x01 (Entry Table index at header[0x00]; each 2-byte entry offsets into animation sequence data holding frame timing, sprite indices, and blend mode), motion_type_flag 0x02 (bit 1 align_sprite_to_velocity, bits 5–7 target anchor mode), animation_target_flag 0x03 (bit 0 spread mode 0=spherical/1=box, bits 1–3 emitter anchor mode, bits 4–7 sprite variant bits), anim_param 0x04 (sprite_variant passed to init_particle_animation), byte_05 unused.** — `[S] 1/3`
  - S: ParticleEmitter core-control table, per `research/key_documents/STRUCTURE_DEFINITIONS.md`
  - src: `research/key_documents/STRUCTURE_DEFINITIONS.md`
- **Emitter flag bytes 0x06/0x07 decode as: bits 0–1 child_death_mode, bits 2–3 child_midlife_mode, bit 4 velocity_inward, bit 5 vestigial (set in 723 emitters but no code reads it), bit 6 color_curve_enable; high byte bits 0–1 homing_arrival_threshold (0=disabled, 1=16, 2=32, 3=48 units) and bit 2 align_to_unit_facing (rotate velocities to caster's facing).** — `[S] 1/3`
  - S: emitter flag-bit tables, per `research/key_documents/STRUCTURE_DEFINITIONS.md`
  - src: `research/key_documents/STRUCTURE_DEFINITIONS.md`
- **Curve-index nibble bytes 0x08–0x0F pair low/high nibbles per parameter: 0x08 emitter_position/particle_spread, 0x09 velocity_base_angle/velocity_direction_spread, 0x0A inertia/(dead code), 0x0B weight/radial_velocity, 0x0C acceleration/drag, 0x0D lifetime/target_offset, 0x0E (unused)/particle_count, 0x0F spawn_interval/homing_strength/homing_adjustment; color-curve indices sit at 0x10 (R low nibble, G high nibble) and 0x11 (B low nibble), each channel independent.** — `[S] 1/3`
  - S: curve-index byte table, per `research/key_documents/STRUCTURE_DEFINITIONS.md`
  - src: `research/key_documents/STRUCTURE_DEFINITIONS.md`
- **Emitter parameters are int16 min/max start/end range pairs: start/end position 0x14–0x1F, spread 0x20–0x2B, velocity base angles 0x2C–0x37, velocity direction spread 0x38–0x43, inertia 0x44–0x4B, weight 0x54–0x5B, radial velocity 0x5C–0x63, acceleration 0x64–0x7B, drag 0x7C–0x93, lifetime 0x94–0x9B, target offset 0x9C–0xAF, particle count/spawn interval 0xB0–0xB7, homing strength 0xB8–0xBF.** — `[S] 1/3`
  - S: emitter field offset tables, per `research/key_documents/STRUCTURE_DEFINITIONS.md`
  - src: `research/key_documents/STRUCTURE_DEFINITIONS.md`
- **Callback parameters 0x4C–0x53 and 0xA8–0xAE are not read by the particle system (it lerps start/end but discards the result); MIPS callbacks read them directly — e.g. E317 CB91 uses 0x4C/0x4E for CLUT blend-mode bits and brightness-table index, and 0xA8/0xAA for trail UV step range (value>>3 = pixels/segment) and ribbon width.** — `[S] 1/3`
  - S: callback_param fields and E317 CB91/CB92 semantics, per `research/key_documents/STRUCTURE_DEFINITIONS.md`
  - src: `research/key_documents/STRUCTURE_DEFINITIONS.md`
- **Child emitters: byte 0xC0 child_emitter_on_death (enabled by emitter_flags_lo & 0x03) and 0xC1 child_emitter_mid_life (enabled by emitter_flags_lo & 0x0C); 0xC2/0xC3 are unused.** — `[S] 1/3`
  - S: child-emitter bytes and enable masks, per `research/key_documents/STRUCTURE_DEFINITIONS.md`
  - src: `research/key_documents/STRUCTURE_DEFINITIONS.md`
- **Target anchor modes (emitter byte 0x02 bits 5–7, where particles home TOWARD): 0x00/0x20 DIRECT_OFFSET (target_offset only), 0x40 CAMERA (camera_position×14 + offset), 0x60 ORIGIN (effect_origin_position + offset), 0x80 TARGET (effect_target_position + offset), 0xA0 CURSOR (cursor_tile×28 + tile_height×−12 + offset); emitter anchor modes (byte 0x03 bits 1–3, where particles spawn FROM): 0x00 WORLD, 0x02 CURSOR, 0x04 ORIGIN (caster), 0x06 TARGET, 0x08 PARENT_PARTICLE, 0x0A CAMERA, 0x0C TRACKED_ENTITY.** — `[S] 1/3`
  - S: anchor-mode value tables, per `research/key_documents/STRUCTURE_DEFINITIONS.md`
  - src: `research/key_documents/STRUCTURE_DEFINITIONS.md`
- **Velocity modes (bytes 0x06–0x07 combined): 0x0000 OUTWARD (standard angular velocity), 0x0010 INWARD (toward emitter center), 0x0400 SKIP (UNIMPLEMENTED — skips velocity calc), 0x0410 OUTWARD_UNIT_ORIENTED (outward rotated to unit facing).** — `[S] 1/3`
  - S: velocity-mode table, per `research/key_documents/STRUCTURE_DEFINITIONS.md`
  - src: `research/key_documents/STRUCTURE_DEFINITIONS.md`
- **PSX semi-transparency blend mode is NOT in the emitter structure — it is baked into the animation sequence word (bit 5 = STP enable, bits 6–7 = ABR: 0=50% average, 1=additive, 2=subtractive, 3=25% additive); the code path at 0x801a59ec computes `tpage = (animation_word & 0xE0) | 0x08`, so particles from the same emitter with different animation indices can render with different transparency.** — `[S] 1/3`
  - S: blend-mode bits and code at 0x801a59ec, per `research/key_documents/STRUCTURE_DEFINITIONS.md`
  - src: `research/key_documents/STRUCTURE_DEFINITIONS.md`

## Notes

(empty — user territory)

## Related

- [[Effect File Format]]
- [[Effect Execution Model]]
- [[Particle Runtime State]]
