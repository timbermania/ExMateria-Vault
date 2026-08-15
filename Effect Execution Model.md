# Effect Execution Model

FFT's runtime execution architecture for E###.BIN visual effects: a per-frame main loop drives a 16-bit opcode script interpreter whose words pack a 9-bit opcode (bits 0–8) and a 7-bit flags field (bits 9–15) and which dispatches through a 512-entry jump table, with two dominant execution patterns — a two-phase timeline-driven animation (opcode 41, op_process_timeline_frame) and a simple single-phase tick (opcode 40, op_animate_tick) — all state held in 248-byte EffectState instances in a fixed array. Emitters are fired by timeline-channel keyframes in Pattern 1 (not by script opcodes directly), feeding the shared particle pipeline (emitter_control_routine / update_all_particles).

## Points

- **The per-frame effect main loop (effect_system_main_loop at 0x801A18D8) iterates the active-effect linked list, executes each effect's script via effect_script_dispatcher (0x801A4CF0) until the script yields (return 0), continues (return 1), or aborts (return 2), then increments frame_counter, which wraps at 160.** — `[S] 1/3`
  - S: effect_system_main_loop 0x801A18D8, effect_script_dispatcher 0x801A4CF0, per `research/key_documents/EFFECT_EXECUTION_MODEL.md`
  - src: `research/key_documents/EFFECT_EXECUTION_MODEL.md`
- **Effect scripts are 16-bit opcodes dispatched through a jump table at 0x801B67C8: the interpreter reads the opcode at script_data_ptr[script_position], masks it with 0x1FF to select a handler index 0–511, and the handler advances script_position by 2, 4, or 6 bytes.** — `[S] 1/3 CONTESTED`
  - S: jump table 0x801B67C8 and 0x1FF opcode mask, per `research/key_documents/EFFECT_EXECUTION_MODEL.md`
  - S: same 0x1FF opcode mask, per `research/key_documents/SCRIPT_EDITOR_LESSONS.md`
  - src: `research/key_documents/EFFECT_EXECUTION_MODEL.md`
- **Pattern 1 (complex two-phase animation) is handled by op_process_timeline_frame (opcode 41, 0x801A4838) and is used by 152 E###.BIN files, including E001 Cure and E019 Fire 4.** — `[S] 1/3`
  - S: op_process_timeline_frame 0x801A4838, per `research/key_documents/EFFECT_EXECUTION_MODEL.md`
  - src: `research/key_documents/EFFECT_EXECUTION_MODEL.md`
- **Pattern 1's ROOT effect progresses through four stages: phase 1 (timeline triggers emitters via keyframes), child spawning (per-target children with configurable delay), phase 2 (resolution animations), and winddown (waits for active particles to die).** — `[S] 1/3`
  - S: per `research/key_documents/EFFECT_EXECUTION_MODEL.md`
  - src: `research/key_documents/EFFECT_EXECUTION_MODEL.md`
- **Pattern 1 child effects are spawned per target during the phase transition, start execution at script offset 0x24, and run op_animate_tick (opcode 40) autonomously rather than op_process_timeline_frame.** — `[S] 1/3`
  - S: child script start offset 0x24, per `research/key_documents/EFFECT_EXECUTION_MODEL.md`
  - src: `research/key_documents/EFFECT_EXECUTION_MODEL.md`
- **Pattern 1's timeline carries 5 particle, 4 color, and 3 sound channels plus a camera track.** — `[S] 1/3`
  - S: per `research/key_documents/EFFECT_EXECUTION_MODEL.md`
  - src: `research/key_documents/EFFECT_EXECUTION_MODEL.md`
- **Pattern 2 (simple single-phase animation) is handled by op_animate_tick (opcode 40, 0x801A3408), is used by 136 files, tracks 5 particle + 3 sound + 4 color channels, and tracks progress via the anim_progress counter.** — `[S] 1/3`
  - S: op_animate_tick 0x801A3408, per `research/key_documents/EFFECT_EXECUTION_MODEL.md`
  - src: `research/key_documents/EFFECT_EXECUTION_MODEL.md`
- **EffectState's common fields sit at fixed offsets — next_effect_index 0x00, script_position 0x06, script_data_ptr 0x08, child_effect_indices[4] 0x0C, active_particle_count 0x1C, frame_counter 0x20, particle_list_head 0xD0 — with Pattern 1 fields occupying 0x28–0xCF and Pattern 2 fields 0x28–0x67.** — `[S] 1/3`
  - S: per `research/key_documents/EFFECT_EXECUTION_MODEL.md`
  - src: `research/key_documents/EFFECT_EXECUTION_MODEL.md`
- **Tracked child effects are spawned by opcode 2 (op_spawn_child_effect) into up to 4 child_effect_indices slots and referenced by opcodes 3, 26, and 27, whereas fire-and-forget children spawned by process_timeline_frame during the phase transition are unlimited and allocated from the global EffectState pool.** — `[S] 1/3`
  - S: opcodes 2/3/26/27 and child_effect_indices, per `research/key_documents/EFFECT_EXECUTION_MODEL.md`
  - src: `research/key_documents/EFFECT_EXECUTION_MODEL.md`
- **In Pattern 1, emitters are not started directly by script opcodes: op_process_timeline_frame reads timeline channel data each frame, the channels specify when each emitter spawns particles via keyframes, and emitters fire via emitter_control_routine (0x801A634C).** — `[S] 1/3`
  - S: emitter_control_routine 0x801A634C, per `research/key_documents/EFFECT_EXECUTION_MODEL.md`
  - src: `research/key_documents/EFFECT_EXECUTION_MODEL.md`
- **The timeline section (at header offset 0x1C) holds a header (phase1_duration, spawn_delay, phase2_offset), 5 particle channels per phase of 128 bytes each, and color/sound/camera tracks.** — `[S] 1/3`
  - S: timeline section at header[0x1C], per `research/key_documents/EFFECT_EXECUTION_MODEL.md`
  - S: channel composition (5 particle + 4 color + 3 sound per phase; camera as 3 tables × 3 tracks), per `research/key_documents/EFFECT_FILE_FORMAT.md`
  - src: `research/key_documents/EFFECT_EXECUTION_MODEL.md`
- **Opcode 38 (op_spawn_emitter) calls emitter_control_routine (0x801A634C) to read an emitter template from the Particle System section, interpolate its parameters using animation curves, and spawn particles into particle_list_head.** — `[S] 1/3`
  - S: emitter_control_routine 0x801A634C, per `research/key_documents/EFFECT_EXECUTION_MODEL.md`
  - src: `research/key_documents/EFFECT_EXECUTION_MODEL.md`
- **Opcode 37 (update_all_particles, 0x801A2EB4) advances all particles each frame: it calls integrate_particle_motion (0x801A9BB0) for physics, processes lifetime countdown / animation-driven death, and spawns child emitters on death or mid-life.** — `[S] 1/3`
  - S: update_all_particles 0x801A2EB4, integrate_particle_motion 0x801A9BB0, per `research/key_documents/EFFECT_EXECUTION_MODEL.md`
  - src: `research/key_documents/EFFECT_EXECUTION_MODEL.md`
- **The effect subsystem's global pointers are 0x801BBF78 (sprite_def_table_ptr), 0x801BBF7C (effect_anim_tbl_ptr), 0x801BBF84 (timeline_channel_base), 0x801BBF88 (effect_data_ptr), 0x801BBF8C (animation_table_ptr), 0x801BC0C8 (timeline_section_ptr), 0x801BACC8 (effect_flags_ptr), and 0x801B9258 (time_scale_ptr).** — `[S] 1/3`
  - S: global pointer addresses 0x801BBF78–0x801B9258, per `research/key_documents/EFFECT_EXECUTION_MODEL.md`
  - S: 0x801BC0C8 (timeline_section_ptr), per `research/key_documents/LUA_DEBUGGING.md`
  - S: same pointer set with header-field provenance, per `research/key_documents/EFFECT_FILE_FORMAT.md`
  - src: `research/key_documents/EFFECT_EXECUTION_MODEL.md`
- **EffectState slot 1's timeline_frame_counter sits at 0x801BF14C (offset 0x28 into the slot, whose base is 0x801BF124).** — `[S] 1/3`
  - S: 0x801BF14C (slot 1 timeline_frame_counter), per `research/key_documents/LUA_DEBUGGING.md`
  - src: `research/key_documents/LUA_DEBUGGING.md`
- **Four further effect-subsystem globals are mapped: 0x801BBF74 (g_sound_section_ptr, feds), 0x801BC0DC (g_sound_data_base), 0x801BAD0C (g_effect_context), and 0x801B48D0 (effect_base_lookup_table).** — `[S] 1/3`
  - S: addresses 0x801BBF74/0x801BC0DC/0x801BAD0C/0x801B48D0, per `research/key_documents/LUA_DEBUGGING.md`
  - src: `research/key_documents/LUA_DEBUGGING.md`
- **The upper 7 bits (bits 9–15) of an effect-script 16-bit word form a flags field: flags = (word >> 9) & 0x7F, complementing the 9-bit opcode in bits 0–8.** — `[S] 1/3`
  - S: word layout [FLAGS:7 bits][OPCODE:9 bits], per `research/key_documents/SCRIPT_EDITOR_LESSONS.md`
  - src: `research/key_documents/SCRIPT_EDITOR_LESSONS.md`
- **Effect-script instruction sizes are fixed per opcode — 2 bytes (no args): opcodes 3,4,5,9,10,12,13,15,32,33,36,37,38,39,40,42,43,44,45; 4 bytes (1 arg): opcodes 0,1,2,6,7,16,26,27,29,30,31,34,35,41; 6 bytes (2 args): opcodes 17,18,19,20,21,22,23,24,25,28; 8 bytes (3 args): opcodes 8,11,14 — each instruction being a 16-bit word followed by (size−2)/2 16-bit args.** — `[S] 1/3 CONTESTED`
  - S: per-opcode instruction size table, per `research/key_documents/SCRIPT_EDITOR_LESSONS.md`
  - src: `research/key_documents/SCRIPT_EDITOR_LESSONS.md`

## Notes

(empty — user territory)

## Related

- [[E001.BIN Memory Mapping]]
- [[Color Track Interpolation]]
- [[Effect File Format]]
