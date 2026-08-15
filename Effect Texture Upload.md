# Effect Texture Upload

Verified procedure for staging and uploading custom-effect textures (e.g. 3D spheres) from a PCSX-Redux Lua session: palette and pixel data are staged in the Effect Buffer at 0x801C2500, pixel data is pushed to VRAM with LoadImage (FUN_800248fc) after a DrawSync, indexed-color CLUTs go through the game's upload_clut (FUN_800926d8) shadow-buffer path, and calls into FFT code use hardcoded JAL encodings; the on-disk texture section of an effect file is laid out as palette 1, palette 2, and 8-bit pixel indices at +0x404, and E001's extracted texture BMP maps to texture UVs with a fixed (+6, +24) offset.

## Points

- **Custom effect texture data is staged in the Effect Buffer region 0x801C2500–0x801DF000 — palette 1 at 0x801C2500 (512 B, 256 colors × 2 B BGR555), palette 2 at 0x801C2700 (512 B), pixel data at 0x801C2900 (~115 KB) — which is unused during the spell charge animation before the effect image loads.** — `[S·D] 2/3`
  - S: Effect Buffer addresses 0x801C2500/0x801C2700/0x801C2900/0x801DF000, per `research/key_documents/PCSX_BRIDGE.md`
  - D: test_working_sphere.sh / test_textured_sphere.sh probes (doc 2026-04-16)
  - src: `research/key_documents/PCSX_BRIDGE.md`
- **All VRAM uploads (texture pixels and CLUT palettes) must go through LoadImage (FUN_800248fc); FUN_800926d8 is a palette buffer copy, not a VRAM upload.** — `[S·D] 2/3`
  - S: symbols FUN_800248fc (LoadImage), FUN_800926d8, per `research/key_documents/PCSX_BRIDGE.md`
  - D: test_textured_sphere.sh probe (doc 2026-04-16)
  - src: `research/key_documents/PCSX_BRIDGE.md`
  - ⚠ SUPERSEDED (2026-08-14) by: Indexed-color palette uploads in embedded effect code must use the game's upload_clut (FUN_800926d8) shadow-buffer path (DAT_800e4ea4, flag DAT_800995ec=1) — LoadImage does not take effect for indexed-color CLUTs (it still works for pixel data).
- **DrawSync(0) at 0x800246d4 must be called before a VRAM upload to wait for pending GPU operations to finish.** — `[S·D] 2/3`
  - S: DrawSync address 0x800246d4, per `research/key_documents/PCSX_BRIDGE.md`
  - D: test_textured_sphere.sh probe (doc 2026-04-16)
  - src: `research/key_documents/PCSX_BRIDGE.md`
- **LoadImage takes a 4×int16 RECT {x, y, w, h}; verified upload geometry is CLUT row 1 (0, 492, 256, 1) 512 B, CLUT row 2 (0, 493, 256, 1) 512 B — VRAM (0, 492–493) corresponds to CLUT=0x7B00 — and a 256×256 texture (384, 0, 128, 256) 65536 B, because 8-bit indexed textures pack 2 pixels per VRAM halfword (VRAM width = pixel width / 2).** — `[S·D] 2/3`
  - S: LoadImage RECT usage at 0x800248fc, per `research/key_documents/PCSX_BRIDGE.md`
  - D: test_textured_sphere.sh probe (doc 2026-04-16)
  - src: `research/key_documents/PCSX_BRIDGE.md`
- **JAL instructions to 0x80xxxxxx FFT code must be encoded as 0x0C000000 | ((target & 0x0FFFFFFF) >> 2); the naive 0x0C000000 + target/4 leaves bit 25 set and corrupts the JAL opcode (000011) into BGTZ (000111), yielding a `bgtz $zero, offset` that never branches — e.g. jal 0x800248fc = 0x0C00923F, jal 0x800246d4 = 0x0C0091B5.** — `[S·D] 2/3`
  - S: target addresses 0x800248fc/0x800246d4, per `research/key_documents/PCSX_BRIDGE.md`
  - D: test_textured_sphere.sh probe (doc 2026-04-16)
  - src: `research/key_documents/PCSX_BRIDGE.md`
- **Indexed-color effect palette uploads must use the game's upload_clut (FUN_800926d8, JAL 0x0C0249B6; args palette_ptr, clut_line, sub_palette_idx, full_upload), which writes the shadow buffer DAT_800e4ea4[clut_line×0x100+idx] (via FUN_80092620) and sets DAT_800995ec=1 so the actual VRAM transfer happens later in the rendering pipeline — LoadImage (FUN_800248fc) does not take effect for the CLUT (it still works for pixel data) — with effect CLUT line 0x0C → VRAM (0, 492) → CLUT 0x7B00 (Palette 1) and 0x0D → VRAM (0, 493) → CLUT 0x7B40 (Palette 2).** — `[S] 1/3`
  - S: symbols FUN_800926d8, FUN_80092620, DAT_800e4ea4, DAT_800995ec, per `research/key_documents/CUSTOM_EFFECT_HOOKS.md`
  - src: `research/key_documents/CUSTOM_EFFECT_HOOKS.md`
- **For 8-bit indexed textures (TPAGE TP=1), texture U = (VRAM_X − TPAGE_base_X) × 2 and V = VRAM_Y (no multiplier on Y); texture page width is 256/128/64 pixels in 4/8/16-bit mode respectively (each 16-bit VRAM halfword holds 2 pixels in 8-bit) — verified on the E001 star: VRAM (391, 95) with TPAGE 0x00A6 (base X = 6×64 = 384) gives UV (14, 95).** — `[D] 1/3`
  - D: E001 star UV verification in the PCSX-Redux VRAM viewer (doc 2026-04-16)
  - src: `research/key_documents/CUSTOM_EFFECT_HOOKS.md`
- **CLUT field encoding (16-bit, at primitive offset 0x0E): bits 0–5 = VRAM X ÷ 16, bits 6–14 = VRAM Y (e.g. VRAM (0, 492) = (492 << 6) | 0 = 0x7B00); TPAGE field encoding (16-bit, at offset 0x1A): bits 0–3 = TX (page X = TX×64), bit 4 = TY (page Y = TY×256), bits 5–6 = ABR (semi-transparency mode 0–3), bits 7–8 = TP (color mode: 0=4-bit, 1=8-bit, 2=15-bit) — e.g. an 8-bit texture at VRAM (384, 0) with ABR=1 gives TX=6, TPAGE = 6 + 0 + 32 + 128 = 0x00A6.** — `[ ] 0/3`
  - src: `research/key_documents/CUSTOM_EFFECT_HOOKS.md`
- **The on-disk texture section of an effect file (pointed to by the header's texture pointer at offset 0x28) is laid out as palette 1 (512 B = 256 × 16-bit colors) at +0x000, palette 2 (512 B) at +0x200, and 8-bit pixel indices (variable size) at +0x404.** — `[S] 1/3`
  - S: texture section layout at header pointer 0x28, per `research/key_documents/CUSTOM_EFFECT_HOOKS.md`
  - src: `research/key_documents/CUSTOM_EFFECT_HOOKS.md`
- **E001.BIN's extracted texture BMP maps to texture UVs with a fixed offset — U = BMP_X + 6, V = BMP_Y + 24 (the offset varies per effect file and must be calibrated against a feature known in both BMP and VRAM) — verified on the star: BMP (18, 95) → UV (24, 119), matching the VRAM-derived (396 − 384) × 2, 119.** — `[D] 1/3`
  - D: E001 star BMP↔UV cross-check against the PCSX-Redux VRAM viewer (doc 2026-04-16)
  - src: `research/key_documents/CUSTOM_EFFECT_HOOKS.md`

## Notes

(empty — user territory)

## Related

- [[E001.BIN Memory Mapping]]
- [[Effect Execution Model]]
