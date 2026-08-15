# Ordering Table & AddPrim

How custom effect primitives are submitted to the PSX GPU: the ordering table (OT) is an array of per-depth linked-list heads (24-bit pointers) whose base address is held in g_ordering_table_ptr (0x801BC0CC), and AddPrim (0x80023BB4) links each primitive into a depth chain at the head; depth values control occlusion — higher depth renders behind, lower in front — so the hook computes a depth from the caster's screen Y (screen_y>>2 + 115, empirically tuned) to place effects at the caster's depth.

## Points

- **The ordering table is an array of linked-list heads, one per z-depth, each entry a 24-bit pointer to the first primitive at that depth; the OT base is read from g_ordering_table_ptr at 0x801BC0CC, and the OT entry for a depth is OT_base + depth×4.** — `[S] 1/3`
  - S: g_ordering_table_ptr 0x801BC0CC, per `research/key_documents/CUSTOM_EFFECT_HOOKS.md`
  - src: `research/key_documents/CUSTOM_EFFECT_HOOKS.md`
- **OT depth semantics (verified empirically): higher depth renders BEHIND, lower depth renders IN FRONT, because the GPU processes OT entries from highest index to lowest — depth 0 renders last (in front of everything) and depth 200 first (behind everything); depth ≈ screen_y>>2 approximately matches unit sprite depth.** — `[S·D] 2/3`
  - S: g_ordering_table_ptr 0x801BC0CC (the OT it points to), per `research/key_documents/CUSTOM_EFFECT_HOOKS.md`
  - D: empirical OT depth test (depth 0 vs 200 vs screen_y>>2) (doc 2026-04-16)
  - src: `research/key_documents/CUSTOM_EFFECT_HOOKS.md`
- **AddPrim (0x80023BB4; a0 = OT entry, a1 = primitive) inserts at the head of the depth chain (*(uint*)prim = *ot_entry; *ot_entry = prim & 0x00FFFFFF), so primitives at the same depth render in reverse order of AddPrim calls (last added = first rendered = backmost); to place an effect at roughly the caster's depth, the hook uses depth = (screen_y >> 2) + 115, with the +115 offset found by binary search (120 placed the effect behind all units, 100 in front of units in front of the caster).** — `[S·D] 2/3`
  - S: AddPrim 0x80023BB4, per `research/key_documents/CUSTOM_EFFECT_HOOKS.md`
  - D: +115 caster depth offset found by binary search (doc 2026-04-16)
  - src: `research/key_documents/CUSTOM_EFFECT_HOOKS.md`

## Notes

(empty — user territory)

## Related

- [[PSX GPU Primitives]]
- [[Embedded MIPS Effect Code]]
- [[Custom Effect Hooks]]
