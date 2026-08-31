# Consensus-Protocol/cmp170hx wiki — imported reference material

Source: https://github.com/Consensus-Protocol/cmp170hx (docs tree), cloned 2026-08-31.
Imported verbatim as third-party reference; see their methodology.md for sourcing rules.

- register-index.md       — appendix, all registers indexed by address (~100 entries in 0x0082xxxx FUSE block)
- register-reference.md   — unlock/, the full register reference with per-block tables
- fuses-and-otp.md        — hardware/, the 121-register cross-card fuse survey (15 Ampere cards)
- 80gb.md                 — frontier/, community 80 GB attempt history (never passed ~40 GB)
- methodology.md          — appendix, how the wiki sources and rates its claims
- artifacts.md            — appendix, artifact inventory
- external-sources.md     — appendix, external source list

Known contradictions with our repo (flagged in 04_h2d-gsp-access-path_to-confirm.md §1):
- fuses-and-otp.md caveat says no 8 GB card was probed; register-reference.md line 481
carries a later clean-room read of an 8 GB card (OPT_SKU_ID=0x80). Both are kept verbatim.
- DEVIDB: fuses-and-otp.md corrects the "8 GB device ID" reading to DEVIDA+0x40 mechanical.
