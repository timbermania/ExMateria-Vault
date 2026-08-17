# E046 Blend Map

E046 (DEMI2, the two-part battle effect) per-frameset blend-mode and emitter-binding layout, read authoritatively from the ABR bits of each frame's `tpage_blend` field (0=normal/avg, 1=ADD, 2=SUB, 3=add¼). As of the 2026-07-31 fold audit the map is: seq0/seq1 subtractive (emitters e0/e1 render the big black blob), seq2/seq3 additive (e2/e3 magenta blob, e4 ~1px), and seq4 a big white glow with no emitter bound — so the subtractive particles are e0/e1 and e2/e3/e4 are all additive. The audit's congruent additive target was emitter-2/seq-2 held on frameset 26 (a 36×36 magenta blob) over a flat mid-gray (128,128,128) screen, where `with − without` reduces to `WITH − 128` per channel.

## Points

- **E046 (DEMI2)'s frameset blend modes come from each frame's `tpage_blend` ABR bits (0=normal/avg, 1=ADD, 2=SUB, 3=add¼) — SEQ0 framesets 0–12 SUBTRACTIVE (emitters e0/e1, the big black blob), SEQ1 framesets 13–25 SUBTRACTIVE (no emitters bound), SEQ2 framesets 26–32 ADDITIVE (e2/e3, the magenta blob), SEQ3 framesets 47–50 ADDITIVE (e4, ~1px), SEQ4 framesets 36–46 ADDITIVE with no emitter bound (the big white glow) — so the subtractive particles are e0/e1 and e2/e3/e4 are all additive, correcting the prior "e2 = subtractive cloud / e4 = the gold additive target" notes.** — `[D] 1/3`
  - D: oracle held-particle capture `captures/oracle_emitter2_seq2_fs26_held.{raw,png}` (2026-07-31): emitter-2/seq-2 held on frameset 26 renders as a 36×36 additive magenta blob (white-clamped core + magenta halo falloff)
  - src: `research/working_documents/demi2_fold_audit/README.md`

## Notes

(empty — user territory)

## Related

- [[Display Space Blend Fold]]
- [[PSX GPU Primitives]]
