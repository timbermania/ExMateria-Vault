# FEDS Sound Definition Format

The "feds" sound definition section (header[0x20]) of E###.BIN files does not contain raw audio: it holds SMD-like sequenced instructions interpreted at runtime by the sound engine, with a header (magic, size, pair count, resource ID, data offset, linked-list pointer), a channel offset table readable pair-major by FFT or channel-major by tools, a per-sound-ID env-multiplier table (chan_92_init), and a verified opcode set including the newly identified D0/D1 pitch-bend opcodes used by Cure's "rising shimmer".

## Points

- **The sound definition section (header[0x20], magic "feds") contains SMD-like sequenced instructions, NOT raw VAG/ADPCM samples; it is interpreted at runtime by the sound engine, with a jump table at 0x80028B0C and handlers (0x80015874–0x80016680) that return input_ptr + params consumed.** — `[S] 1/3`
  - S: jump table 0x80028B0C and handler return convention, per `research/key_documents/STRUCTURE_DEFINITIONS.md`
  - src: `research/key_documents/STRUCTURE_DEFINITIONS.md`
- **FEDS header fields: magic "feds" at 0x00, data_size at 0x04, pair_count_plus1 at 0x08 (num_channels = (field−1)×2 — E001 field=3 → 4 channels, E019 field=4 → 6), resource_id at 0x0A, data_offset at 0x0C (FFT reads `lhu v0, 0xc(s5)` at PC 0x80013BEC; upper 16 bits zero across all 512 E*.BIN), list_next_ptr at 0x10 (zeroed per-effect by FUN_801A18EC at PC 0x801A1944; non-zero only for the SCUS-static blocks DAT_80047610/DAT_8004D9B8).** — `[S] 1/3`
  - S: FEDS header table 0x00–0x13, PC 0x80013BEC, FUN_801A18EC PC 0x801A1944, per `research/key_documents/STRUCTURE_DEFINITIONS.md`
  - src: `research/key_documents/STRUCTURE_DEFINITIONS.md`
- **The channel offset table is one memory with two views: pair_record[0] sentinel [0,0] at 0x14 (sid=0; FFT skips emit when sound_id < 2), pair_record[1..N] at 0x18 = [ch_a_offset, ch_b_offset], read pair-major at +0x14 + sid×4 by FFT (FUN_80013B20, PC 0x80013C34–0x80013C40) or channel-major at +0x18 + ch×2 by tools; there is no end marker — channel N's size is offset[N+1] − offset[N] and the last channel runs to data_size.** — `[S] 1/3`
  - S: channel offset table dual view, FUN_80013B20 PC 0x80013C34–0x80013C40, per `research/key_documents/STRUCTURE_DEFINITIONS.md`
  - ⚠ SUPERSEDED (2026-08-19) by: There *is* an end marker — the driver stores `bank + offset` and reads to a `0x90` end bar, with no extent at all; bounding a stream at the next stream's start truncates 94 of the 2,388 streams in the shipped banks, 24 of them in `SOUND/SYSTEM.SED`
  - src: `research/key_documents/STRUCTURE_DEFINITIONS.md`
- **chan_92_init[] sits at data_offset (pair_count_plus1 bytes, one instr_byte per sound ID 0..N) and FFT computes `chan_92 = min((instr_byte × 0x6000) >> 7, 0x7FFF)` (PC 0x80013C04) with a saturation clamp when instr_byte > 0xAA (PC 0x80013C2C); two layout patterns exist — a dedicated block (channel_offsets[0] > data_offset + num_sids, e.g. cure_4) and an overlapping block (e.g. original E001, where the ASCII "PTT" bytes 0x50/0x54/0x54 ARE chan_92_init[0..2] → chan_92 15360/16128/16128).** — `[S] 1/3`
  - S: chan_92_init layout, PC 0x80013C04/0x80013C2C, per `research/key_documents/STRUCTURE_DEFINITIONS.md`
  - ⚠ SUPERSEDED (2026-08-19) by: `0x6000` is not a constant of the formula, it is an argument — the law is `(volume8.8 × instr_byte) >> 7` clamped at `0x7FFF`, and `0x6000` is volume 96 in 8.8, which 6 of the 8 play entrypoints hardcode but the two settings variants (`SCUS_942.21+2ee8`, `+2fb4`, which pass `v << 8`) do not
  - src: `research/key_documents/STRUCTURE_DEFINITIONS.md`
- **The verified FEDS opcode set (param counts from handler returns): Note 00–7F (1 param), Rest 80 (1, 0x80015874), Fermata 81 (1, 0x8001589C), NOP 82 (0, 0x8001586C), EndBar 90 (0, 0x800158F8), Loop 91 (0, 0x800159DC), Octave 94 (1, 0x80015A10), RaiseOctave 95 (0, 0x80015A28), LowerOctave 96 (0, 0x80015A40), Repeat 98 (1, 0x80015AB8), Coda 99 (0, 0x80015B00), Tempo A0 (1, 0x80015CB0), Instrument AC (1, 0x80015DD0), Flag_0x800 B0 (0, 0x80015EA8), ReverbOn BA (0, 0x800160E4), ReverbOff BB (0, 0x80016110), Release C4 (1, 0x800161E0), Dynamics E0 (1, 0x80016614), Expression E2 (1, 0x80016680).** — `[S] 1/3`
  - S: SMD-like opcode table with handler addresses, per `research/key_documents/STRUCTURE_DEFINITIONS.md`
  - ⚠ SUPERSEDED (2026-08-19) by: `E2` takes **2** parameters, not 1 — it is a level ramp, `(units, level)` — so a walker giving it 1 byte desynchronises the rest of the channel; and `C4` is the sustain rate, not release: `C0..CA` is a full ADSR family in which `C4 = sustainRate` and `C5 = releaseRate`
  - src: `research/key_documents/STRUCTURE_DEFINITIONS.md`
- **Pitch bend opcodes D0 SetPitchBend (0x800162B4) and D1 AddPitchBend (0x800162D8) take a 1-byte param that is sign-extended and scaled ×32 (sll 24 / sra 19) and stored to channel +0x86 — D0 sets the value, D1 adds to the current one; Cure uses them for the ascending "rising shimmer" sparkle.** — `[S] 1/3`
  - S: D0/D1 handler disassembly at 0x800162B4/0x800162D8, per `research/key_documents/STRUCTURE_DEFINITIONS.md`
  - src: `research/key_documents/STRUCTURE_DEFINITIONS.md`
- **Resource lookup: resource_id (0x0A) << 16 forms the lookup key in g_sound_data_base (0x801BC0DC, verified at 0x801A1158), and play_sound(address) matches the address's upper 16 bits against registered resource_id fields while walking g_sound_resource_list (0x80032A00) in FUN_80013B20; init_sound_section (0x801A10EC), register_sound_resource (0x80017E7C), unregister_sound_resource (0x80017EB8), g_sound_section_ptr (0x801BBF74).** — `[S] 1/3`
  - S: resource lookup at 0x801A1158, g_sound_resource_list 0x80032A00, runtime globals 0x801BBF74/0x801BC0DC, per `research/key_documents/STRUCTURE_DEFINITIONS.md`
  - src: `research/key_documents/STRUCTURE_DEFINITIONS.md`
- **A voice stream ends on its own `0x90` end bar and not at the next stream's offset: over 2,388 streams in 514 shipped banks, 2,386 end on a `0x90` of their own and 94 run past the next stream's start — every one of them into the same effect's second voice, which begins 2 to 6 bytes inside the first and shares its tail.** — `[S·D] 2/3`
  - S: the driver has no extent — the play path stores `bank + offset` into `ch+0x18` and the sequencer reads until `0x90`
  - D: census over every `.SED` on the disc (web-psx `SedBank.extentOf`, `docs/audio-seam.md` [audio.smd.sed]): 2,388 of 2,388 terminate properly once the boundary is counted, the two apparent exceptions being the last stream of `SOUND/SYSTEM.SED` and of `SOUND/ENV.SED`, whose `0x90` is the bank's final byte; median stream length 34 bytes, longest 218. Bounding at the next start truncates 94, of which 24 are in `SOUND/SYSTEM.SED` (cross-referenced 2026-08-19)
  - src: external contribution — web-psx `docs/audio-seam.md` [audio.smd.sed] (see [[Web-psx Cross-Validation]])
- **The channel level the play path installs is `(volume8.8 × bank_byte) >> 7` clamped at `0x7FFF`, and the worker also sets bit 3 of `ch+0` so that the stream's own note dynamics do not overwrite the level the caller asked for — a music channel never has that bit.** — `[S·D] 2/3`
  - S: `SCUS_942.21+4320` fills the channel in: octave base `0x3c`, gate `0x0f`, dynamics 127, pan from the caller, level from the caller's volume scaled by the bank's per-effect byte
  - D: a scripted boot confirms the volume argument — 349 of 349 effect calls at volume 96 and pan 64 — and the scaling then predicts, from the disc alone, the channel volumes a *different* session was observed holding: `SOUND/SYSTEM.SED` effects 115 and 1 carry bank byte 88 and give 66, effects 60 and 133 carry 100 and give 75 (web-psx `docs/audio-seam.md` [audio.smd.sed], [audio.door.crosses]; cross-referenced 2026-08-19)
  - src: external contribution — web-psx `docs/audio-seam.md` [audio.smd.sed] (see [[Web-psx Cross-Validation]])
- **The `.SED` voice language is the same language as the `.SMD` music sequences, and web-psx's interpreter of it carries about 90 opcodes against the 21 verified here — including families absent from this note entirely: vibrato/tremolo/pan sweep and LFO (`D4`–`DC`, `E3`–`EF`, `F0`–`F7`), `A2` tempo ramp, `A6`/`A7` song volume and its ramp, `FE` waveset, and `99` as the end of a `98` repeat rather than a bare "Coda".** — `[S] 1/3`
  - S: web-psx `src/jukebox/seqfmt.ts` command table and `docs/sequence-format.md` [seqfmt.commands]; `B0` is the phrase tie/hold rather than an opaque flag, and `D0`/`D1` are transpose and transpose-add in eighths of a semitone — the ×32 scaling in this note against the 0x100-per-semitone key table is exactly an eighth, so the two readings agree and only the RAM offset was new to us (2026-08-19)
  - src: external contribution — web-psx `docs/sequence-format.md` [seqfmt.commands] (see [[Web-psx Cross-Validation]])

## Notes

(empty — user territory)

## Related

- [[Effect File Format]]
- [[Web-psx Cross-Validation]]
