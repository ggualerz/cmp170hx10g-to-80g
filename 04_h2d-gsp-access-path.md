# Phase 04 Research Report — H2D/GSP Access Path

> [!WARNING]
>
> ## This research is not finished
>
> The CMP 170HX 10 GB → stable 80 GB problem remains **open**. This report
> documents what has been tried, what worked, what failed, and what remains
> untested. Several conclusions are inferences, not proven facts — these are
> flagged inline as **(inference)** and tracked in
> [`04_h2d-gsp-access-path_to-confirm.md`](04_h2d-gsp-access-path_to-confirm.md). The "80 GB is not achievable" assessment
> reflects exhausted approaches, not a proof of impossibility.

**Goal:** understand why the CMP 170HX 10 GB crashes above 40 GiB, and whether
a stable 80 GB operating configuration is achievable.

**Period:** 2026-08-27 to 2026-08-31  
**Hardware:** NVIDIA CMP 170HX 10 GB (GA100, Samsung HBM, PCI ID 10de:2082, rev a1) via eGPU PCIe/USB4  
**Driver:** 610.43.02 open kernel modules (cmpunlocker-patched)  
**GSP firmware:** `gsp_tu10x.bin` (29 MB — the firmware GA100 **actually loads**, NOT `gsp_ga10x.bin`)

This report supersedes the scattered working-session documents
(`LATEST.md`, `STATUS_CONSOLIDATED.md`, `HANDOFF.md`, `SEC2_*.md`,
`CUMEMFREE_WORKAROUND.md`, `VMM_FREE_PATH_RESULTS.md`, `FREE_NOP_RESULTS.md`,
`POST_REBOOT_PLAN.md`, `TODO-experiments.md`, `previous.md`).

Claims that are inference rather than proven are flagged inline as
**(inference)**. See [`04_h2d-gsp-access-path_to-confirm.md`](04_h2d-gsp-access-path_to-confirm.md) for the full list
of open questions.

---

## Executive summary

The card has 80 GiB of physical HBM and boots into 80 GB geometry. The
40 GB limit appears to be enforced by the GSP-RM firmware, keyed to the
card's hardware identity — the OTP fuse `FUSE_PCIE_DEVIDA = 0x2082` is
the primary suspect **(inference — see TO-CONFIRM §1)**. Three
independent host-side device-ID overrides were all ignored by the
GSP-RM. The crash above 40 GiB is a **HAL dispatch layout mismatch** — a
firmware-internal class-selection bug. An A/B static-info dump on this
10 GB card (40 GB control vs 80 GB crash) showed identical SKU/class
fields (638 of 642 words match), but the 8 GB card (0x20C2 → 64 GB
stable) was not tested — so this is not fully cross-SKU proven.

**What works:** 40 GB stable (production-ready). 79 GiB allocation + SM
kernel fill + verify = ALL CLEAN — achieved by the WPR2/PMA-overlap fix
(`fix/late-pma-wpr2-overlap`) on coherent 80 GB geometry, **without** any
SEC2 patch (run `04-genesis-gpu3-x4-80g-kernelmax-fixwpr2-20260828T124248Z`:
31/31 chunks clean, zero failures, clean OOM on overshoot; teardown Xid
119→154 at ~3687 s, after the verdict). 108+ GiB cumulative H2D through a
bounded buffer. A driver-side FREE→NOP patch enabled full
cudaMalloc+cudaFree at 76 GiB with zero Xids (source-only, never
installed — see §7.1).

**What doesn't work:** context teardown / `cudaFree` above ~39.25–39.50 GiB
triggers a deterministic GSP crash. **H2D writes above the 40 GB limit
NEVER passed** — every H2D attempt into high memory faulted (Xid
119→1→119×2→154). The 40 GB wall was never broken. The earlier claim that
a SEC2 shadow write (0x840a48/0x840a94) eliminated the 40 GB wall is
**not confirmed by preserved raw data** — no preserved run contains both
SEC2 writes and a clean 79 GiB result; the one reproduction attempt (v5
build) crashed during the data phase at ~184 s, and `0x840a94` read back
`0xffffffff`. See §7 and TO-CONFIRM §2.

**Current assessment:** 80 GB is not achievable with any mechanism we
have tested. 40 GB stable is the working ceiling. This is an assessment
of exhausted approaches, not a proof of impossibility — see
[`04_h2d-gsp-access-path_to-confirm.md`](04_h2d-gsp-access-path_to-confirm.md) for what undermines it.

---

## 1. Hardware and software identity

```
GPU:      NVIDIA CMP 170HX 10GB, GA100, PCI ID 10de:2082, rev a1
HBM:      Samsung (MODS: 0x41 / XA2_8HI / 8 Gb/channel / 8-high)
Attach:   eGPU PCIe over USB4 (x4 observed, 2.5 GT/s)
Host:     genesis, kernel 7.0.0-30-generic
Driver:   610.43.02 open kernel modules (cmpunlocker-patched)
CUDA:     13.1 (toolkit) / 13.3 (UMD reported)
GSP fw:   gsp_tu10x.bin (29 MB) — GA100 loads THIS, not gsp_ga10x.bin
SKU tbl:  .fwimage offset 0x23460 in gsp_tu10x.bin: class=9, size_index=10 (40GB)
```

Coherent 80 GB geometry (known-good register set):
```
FBPA_CFG1       = 0x02779000   (tier 0x77 = 4096 MiB/FBPA, 20 FBPAs)
MMU_LMR         = 0x0000028B   (MAG=40, SCALE=11 → 80 GiB)
L2/LTC decode   = 0x10000300   (4 GB/channel)
targetFbBytes   = 0x1400000000
```

---

## 2. Where the 40 GB limit lives

The card physically has 80 GiB of addressable HBM. The limit appears to
be enforced in **firmware**, not in the geometry:

```
OTP fuse (FUSE_PCIE_DEVIDA = 0x2082)
   └─> GSP-RM reads device ID at init (NOT from host RPC; NOT from XVE)
          └─> SKU table lookup in gsp_tu10x.bin @ 0x23460
                 class=9 (CMP), size_index=10 → 40 GB
                    └─> RM-internal allocation tracking structure sized for 40 GB
                          └─> total allocation > ~39.25 GiB → structure overflows
                                 └─> RM heap corrupted (code ptr where data ptr expected)
                                        └─> next FREE/BUS_FLUSH RPC → jalr to garbage → Xid
```

> **Inference:** the chain from OTP fuse to SKU table to 40 GB structure
> is inferred from the failed override attempts and the firmware SKU
> table, not from a traced code path. The GSP-RM ignores all host-side
> device-ID inputs, so *something* hardware-side is the source — but
> whether it is `FUSE_PCIE_DEVIDA` specifically, or another fuse, or a
> combination, is not confirmed. See [`04_h2d-gsp-access-path_to-confirm.md`](04_h2d-gsp-access-path_to-confirm.md) §1.

The GSP-RM is the **only** RM on GA100. `NVreg_EnableGpuFirmware=0` fails
with `0x62` firmware-init error — there is no CPU-RM escape hatch. The
proprietary closed driver cannot be patched the same way as the open
modules. The fight is to make **this** GSP-RM believe the card is 80 GB,
or to fix the corrupted structure it builds.

### The 8 GB → 64 GB reframing

`0x20C2` (8 GB CMP 170HX) → 64 GB is **stable and in production** — same
GA100, same GSP-RM, same driver, same `CFG1=0x02779000` tier. The GSP-RM
handles >40 GB fine on the 8 GB card. The 40 GB cap is **SKU-table-specific
to `0x2082`** (device_class=9, size_index=10) — by inference from the SKU
table; the cap has never been lifted on this card. 80 GB is not a fuse state
that exists for the 10 GB card. This suggests the GSP-RM **can** run
large-FB SKUs (inference — the 40 GB cap on `0x2082` remains intact in
every preserved run; no experiment ever broke it).

---

## 3. What has been PROVEN to work

Backed by preserved raw data, not assertions:

| Capability | Status | Evidence |
|---|---|---|
| Boot into 80 GB geometry (CFG1/LMR/fb_length) | ✅ | `fb_length=0x1400000000` in dmesg |
| `nvidia-smi` reports 81920 MiB | ✅ | live |
| Allocate 79 GiB (19×4 + 12×256 MiB) | ✅ | `hbm_kernel_max`, `kernelmax-fixwpr2` run |
| SM-kernel fill + verify 79 GiB, ALL CLEAN | ✅ | run `04-...-kernelmax-fixwpr2-20260828T124248Z` — **WPR2-fix only, no SEC2 patch** |
| Clean OOM on overshoot (WPR2/PMA overlap fix) | ✅ | `cmpunlocker-late-pma-wpr2-bug.md`, fix `e093717` |
| PRAMIN walk: 80 distinct GiB physically present | ✅ | `80gb.md` (community report) |
| H2D 108+ GiB cumulative through bounded buffer — into LOW memory only (chunk 0, below 40 GiB) | ✅ | `b1reserve` run. H2D above the 40 GB limit NEVER passed |
| GSP survives entire data phase at 79 GiB | ✅ | `kernelmax-fixwpr2` run |
| 40 GB stable (production-ready) | ✅ | `gpu_burn` clean, fold tests clean |

---

## 4. The crash, precisely

### Crash mechanism

The GSP-RM builds an internal allocation-tracking structure sized from the
SKU table entry (40 GB). When total allocation crosses ~39.25–39.50 GiB, a
field at offset `+0xB98` in that structure gets filled with a **code
pointer** (`0x5af8818`, in the rm.elf CODE segment) instead of a data
pointer.

```
crash @ 0x5b2b93c:  sw a0, -0x7fc(a5)
  a5 = *(0x43894f8)->field_0x1a8->offset_0xB98 + 0x111000
  *(a5+0xB98) = 0x5af8818  (CODE ptr — the bug; should be a DATA ptr)
  store target = 0x5c0901c  (PH[15] BSS, read-only) → page fault
  mepc=0x5b2b940, mcause=2, mbadaddr=0x5c0901c
```

The store value is a **performance counter** (`divu a0, a0, 1000000` —
Hz→MHz), so the corrupted object is a perf-monitoring structure, not
allocation tracking directly. The global `0x43894f8` is BSS, zeroed at
init by `0x4c133c4`, repopulated later.

### Crash classification: HAL dispatch layout mismatch (Lead C)

Firmware RE on `gsp_tu10x.bin` (the firmware GA100 actually loads) established:

1. **`gsp_tu10x.bin` is a single ELF** (31 program headers). Runtime VAs
   are link VAs — **no LOAD_BIAS to compute**. Crash VAs decode directly:
   `file_off = VA - 0x4bf7000 + 0xbf7000` (PH[25]). (Previous bias work
   targeted `gsp_ga10x.bin` — the wrong file.)

2. The global `G = *(0x43894f8)` is set by **exactly one** non-zero site:
   `0x4c12c08: sd s3, 0x4f8(s6)` inside constructor `0x4c12ba4` (`s3 = a0`).
   The only other write is the zero-store destructor `0x4c133c4`.

3. The crash function `0x5b2b724` reads `base = *(G + 0x1a8)`, then
   `P = *(base + 0xB98)`, then stores a perf-counter at
   `P + 0x111000 - 0x7fc`. `base` lands in CODE → `*(base+0xB98)` is
   garbage → fault.

4. **`+0x1a8` is a HAL method-pointer slot**, not a data pointer. Three
   dispatch-table-fill sites install CODE-segment addresses there (gated
   by a class/engine range check): `0x4c007cc`, `0x4c00cec`, `0x4c087a0`.
   Both the `< 0x210b0` and `>= 0x210b0` paths install the SAME code
   pointer `0x5877fd8` at `+0x1a8`.

5. The crash is a **wrong-method-dispatch**: method `0x5b2b724` is
   dispatched on object G but expects a layout where `+0x1a8` is a data
   pointer. Not a corrupted value — a field interpretation mismatch.

6. The "bad pointer" `0x5af8818` is **not a literal anywhere** in the
   firmware and **not a function start**. It is `*(base+0xB98)` read from
   a code-range address, i.e. garbage.

7. The dispatch filler `0x4c00288` reads its class index as
   `a4 = *(a5 + 0x6a8)` — the object's own NVOC class-ID slot. `+0x6a8`
   is loaded from memory (no small-immediate installs in the dispatch
   range `0x12010..0x2e0d0`); 41 sites install large vtable-class
   pointers for other object types.

8. Both the crash method `0x5b2b724` and constructor `0x4c12ba4` are NOT
   built by any `lui+addi` and NOT present as literals — confirmed
   load-time relocation (radix3). Static vtable hunting cannot pin the
   dispatch slot.

### Crash triggers

The crash manifests two ways depending on trigger:

- **H2D-into-high** (reproduced locally): fn 76 `BUS_FLUSH_WITH_SYSMEMBAR`
  (`0x20800a70`) timeout, sometimes Xid 1. Fires after ~38 GiB cumulative
  H2D with 38+ GiB total allocation. The H2D must touch high-VA mappings
  (B1-RESERVE proved: 40 GiB alloc + H2D into chunk 0 only = clean data,
  but teardown still crashes).

- **Teardown** (always): fn 76 timeouts → Xid 154 PF FLR. The corrupted
  state is created by **allocation alone** — teardown reaches it without
  any H2D into high mappings.

### Teardown crash threshold (bisected)

B1-CROSS bisection — allocate X GiB (256 MiB chunks, no H2D into them),
1 GiB H2D into chunk 0, clean exit, watch teardown for 150s:

| alloc GiB | chunks | teardown |
|---|---|---|
| 8 | 32 | CLEAN |
| 20 | 80 | CLEAN |
| 30 | 120 | CLEAN |
| 36 | 144 | CLEAN |
| 38 | 152 | CLEAN |
| 39.00 | 156 | CLEAN |
| 39.25 | 157 | CLEAN |
| 39.50 | 158 | **FAULT** |
| 39.75 | 159 | FAULT |
| 40.00 | 160 | FAULT |

**Threshold: between 157 chunks (39.25 GiB) and 158 chunks (39.50 GiB).**
This is ~40 GiB minus CUDA context overhead, consistent with a structure
sized for `targetFbBytes=0x0A00000000`.

### The crash is deterministic

Two independent crash dumps (pageable + pinned H2D) are **byte-identical**:
same `mepc`, `mbadaddr`, `a5`, `ra`, `s0`–`s7` — only `sp` differs
slightly (call-depth noise). Random HBM retention decay cannot produce
byte-identical crash dumps. This is a software bug that writes the same
wrong function-pointer value every time.

---

## 5. A/B static-info dump — "wrong stuff" hypothesis DISPROVED

A driver patch (`inject_staticinfo_dump.py`) hexdumped the full
`GspStaticConfigInfo` (returned by GSP-RM via `GET_GSP_STATIC_INFO`) and
`GspSystemInfo` (sent via `SET_GUEST_SYSTEM_INFO`) in both a 40 GB control
boot and an 80 GB crash boot. See `ab_dump/ANALYSIS.md` for the full diff.

**638 of 642 words are identical.** The only differences are FB region
descriptors, `fb_length`, WPR layout, a per-boot PCI BAR address, and one
link-cap bit. **Every SKU/identity/class field is identical:**
`gidInfo`, `SKUInfo` (BoardID, chipSKU, project, CDP, businessCycle),
`engineCaps[]`, all `bIs*` brand flags, `sriovCaps`, `poisonFuseEnabled`.

**Conclusion:** the GSP-RM computes the same SKU/brand/class identity in
both configs. The bad dispatch is NOT selected by any field in the static
config we send or receive. The class ID at `obj+0x6a8` is set from a
class table INSIDE the GSP-RM, indexed by `fb_length`/region geometry, not
by a host-visible identity field. There is no host-side static-config
field to flip that makes 80 GB geometry use the 40 GB-compatible dispatch
layout while keeping `fb_length=80GB`.

**The "we send wrong stuff" hypothesis is disproved.** We are sending the
correct identity; the GSP-RM derives the bad class from the (correct) 80 GB
geometry itself. The bug is firmware-internal.

---

## 5.1 The WPR2/PMA overlap bug (cmpunlocker)

The `late-pma.patch` in the cmpunlocker driver registers a "free" PMA
(Physical Memory Allocator) region so CUDA can use more memory. It
searches for the topmost reserved, non-internal-heap FB region with
`limit >= 8 GiB` and registers it as allocatable.

On the coherent 80-GiB geometry, the topmost reserved region is:

```
region[6]: base=0x13f7100000  limit=0x13ffffffff  (143 MB, reserved)
```

But this region contains WPR2 (the GSP's own protected heap):

```
WPR2:       0x13f7200000 – 0x13fff00000  (141 MB)
RM heap:    0x13f7300000, heapSize=0x7000000 (112 MB)
```

The patch registered the entire 143 MB as free PMA memory — but ~141 MB
of that was actually the GSP's own heap, not free memory at all. Only
~2 MB was genuinely free (1 MB below WPR2 start, 1 MB above WPR2 end).

### Consequences

1. **Phantom capacity:** PMA reported ~141 MB more free memory than
   actually existed.
2. **WPR2 collision:** once the main PMA region (~79 GiB) was
   exhausted, client allocations would receive physical pages inside
   WPR2 — overlapping the GSP-RM's own code, data, and heap objects.
3. **Silent corruption or hardware fault:** writes to WPR2 pages are
   blocked by hardware protection (WPR), causing CE errors or
   corrupting adjacent GSP state.

This is a **>79 GiB hazard**, not the cause of the ~38–40 GiB crash.
`pmaSelector` orders regions by performance (fastest-first), so the
late region is only used at exhaustion. The 76 GiB batched run was
clean; the 38 GiB H2D fault has a separate cause.

### Fix

Added a clamp in `late-pma.patch`: before registering the candidate
region with PMA, check if it overlaps WPR2 (by reading
`gspFwWprStart`/`gspFwWprEnd` from the KernelGsp WPR metadata). If it
overlaps:

- Clamp the registered PMA range to below `gspFwWprStart` (only the
  ~1 MB genuinely free below WPR2).
- Leave the FB region table entry reserved (don't flip
  `bRsvdRegion = NV_FALSE`) so RM heap paths can't allocate the
  overlapping range either.

The fix was built, installed, and validated (`fix/late-pma-wpr2-overlap`,
commit `e093717`): the boot log confirmed the clamp fires, and the 79-GiB
kernel test completed ALL CLEAN with the overshoot probe returning clean
OOM instead of phantom pages.

This bug is **separate from** the 40 GB GSP-RM allocation limit. It is a
cmpunlocker bug, not a firmware bug. Full analysis in
[`cmpunlocker-late-pma-wpr2-bug.md`](cmpunlocker-late-pma-wpr2-bug.md).

---

## 6. Dead ends (do not retry)

| # | Approach | Why dead | Evidence |
|---|---|---|---|
| 1 | WPR2 livepatch via `memdescMapInternal` | MPU blocks host FB mapping of WPR2 → NULL | `gsp-analysis.md` |
| 2 | WPR2 livepatch via BAR0 PRAMIN | MPU → `0xBAD0ACxx` bad-access markers | `gsp-analysis.md` |
| 3 | Pre-load fwimage patch (pImageData before Booter DMA) | Booter verifies image hash even with RSA bypassed → `0xffff` | `sku_fwimage_patch.c` run |
| 4 | FEAT_OVR multi-bit device-ID override | bit-mask enable/disable only, can't reach `0x14` (needs bit 4, read-only) | `SEC2_SHADOW_BIGPICTURE.md` |
| 5 | RPC `PCIDeviceID` override `0x20B50000` | GSP-RM ignores RPC value | `gsp-analysis.md` |
| 6 | RPC `PCIDeviceID` override `0x20B510DE` (correct vendor) | GSP-RM ignores RPC value | `gsp-analysis.md` |
| 7 | `NV_XVE_ID` register write (`0x20B510DE`) | GSP-RM ignores XVE register | `gsp-analysis.md` |
| 8 | PCI config mirror disable | GSP-RM reads fuse directly | `gsp-analysis.md` |
| 9 | SEC2 shadow `0x008411e8` (size_index) write | read-only latched, readback stays `0x0a` | `SEC2_REGISTER_ANALYSIS.md` |
| 10 | FEAT_OVR `0x0082382c` write to `0x14` | only bits 0–3 writable, bit 4 read-only | `SEC2_SHADOW_BIGPICTURE.md` |
| 11 | `FUSE_PCIE_DEVIDA` write at boot (PLM window) | **OTP, rejected at boot too**: `before=0x2082 after=0x2082` | live journalctl |
| 12 | Broad-stroke `0x14` to `0x840a4c–0x840a8c` | did NOT fix fn 76 | commit `4680556` |
| 13 | `0x8411f8`/`0x8411fc` as 80 GB config holders | read `0x00000000` on clean boot; `0x14000000` was state residue | live journalctl |
| 14 | `NVreg_EnableGpuFirmware=0` (CPU-RM) | GA100 hard-requires GSP-RM; `0x62` firmware-init | consensus docs |
| 15 | Proprietary closed driver | unpatchable boot path; not the same insertion points | consensus docs |
| 16 | SEC2 shadow `0x840a48`/`0x840a94` boot writes | claim of "once eliminated the 40 GB wall" not supported by preserved raw data (misattributed WPR2-fix run; v5 repro crashed in data phase; `0x840a94` readback `0xffffffff`) | re-examination of run logs |
| 17 | `cuMemFree` "repairs state" | both `cuMemFree` and `cudaFree` dispatch the SAME RPC (`NV_ESC_RM_FREE` → fn 10); the repair observation was under a different build state | source check |
| 18 | FREE→38 redirect (`FREE_VIDMEM_VIRT`) | fn 38 is a no-op inline on this path, not cuMemFree's handler | source check |
| 19 | Booter ROP FB write (Lead D) | public gadget vocabulary has no FB-write gadget — only `reg_write_indirect` | gadget vocabulary check |
| 20 | CPU-RM / closed driver | GA100 hard-requires GSP-RM; `0x62` firmware-init | consensus docs |
| 21 | VMM API as usable path | clean for `cuMemRelease` at 76 GiB, but `nvidia-smi` after triggers Xid 119+154 — corrupted GSP state persists | `VMM_FREE_PATH_RESULTS.md` |
| 22 | BUS_FLUSH RPC skip (function 76) | function 76 is `GSP_RM_CONTROL` — the generic RM control channel. Skipping = useless GPU. Skipping one cmd = silent corruption | source check |
| 23 | Pre-load fwimage patch (re-confirmed) | Booter verifies image hash even with RSA bypassed → `0xffff` | re-confirmed |
| 24 | WPR2 host livepatch (re-confirmed) | MPU blocks `memdescMapInternal` (NULL) and PRAMIN (`0xBAD0ACxx`) | re-confirmed |

---

## 7. The SEC2 shadow write — claim not confirmed

An earlier session document (`SEC2_SHADOW_OVERRIDE_RESULTS.md`,
2026-08-29) claimed that writing `0x02`/`0x08` to `0x00840a48` and
`0x00840a94` at boot time (before GSP-RM init) eliminated the 40 GB wall
once, and attributed the 79 GiB clean result to the SEC2 patch.
**Re-examination of the preserved raw data does not support this:**

- No preserved run contains both SEC2 shadow writes and a clean 79 GiB
  result. The only preserved raw run matching that output
  (`04-...-kernelmax-fixwpr2-20260828T124248Z`) **predates the SEC2
  patch**, contains no SEC2_SKU / SEC2_DEBUG_SKU_SHADOW lines in its
  kernel log, and used the `fix/late-pma-wpr2-overlap` patch. The
  results document appears to have misattributed the WPR2-fix run's
  result to the SEC2 patch.
- The one reproduction attempt this session (v5 build:
  0x840a48 + 0x840a94 + Lead B) ran `hbm_kernel_max` and **crashed at
  ~184 s — during the data phase**. That is worse than the WPR2-fix run,
  which completed the data phase clean. The SEC2 writes did not
  reproduce the 79 GiB result.
- `0x840a94` read back `0xffffffff` on reproduction attempts (the write
  appears rejected or state-locked).

**What is observed about the registers themselves:**
- `0x00840a48` and `0x00840a94` change values between reads spaced
  seconds apart. `0x840a48` was observed incrementing `0x09 → 0x0a → ...`
  over ~16 seconds. `0x840a94` jumps erratically.
- On one occasion, 44 GiB fill crashed at 839 seconds after boot; on a
  fresh boot (< 60 seconds), 52 GiB fill+verify worked fine.

**What is inference:**
- The "PRNG state" label. The registers change at runtime and the
  boot-time write effect is non-deterministic. Calling them "PRNG state"
  is one explanation — they could also be counters, state machines, or
  registers with a host/GSP race condition. The
  `SEC2_SHADOW_BIGPICTURE.md` document itself flagged this as
  **UNPROVEN**. See [`04_h2d-gsp-access-path_to-confirm.md`](04_h2d-gsp-access-path_to-confirm.md) §2.

**Status:** the SEC2 shadow path is, at best, an unconfirmed timing
lottery. The 79 GiB breakthrough that actually exists is the
**WPR2-fix run**, and it is reproducible from geometry + WPR2 fix alone
(no SEC2 writes).

### 7.1 FREE→NOP driver patch

A driver-side patch was built to test whether the crash is only in the
FREE path or also in other teardown RPCs:

- `inject_free_skip.py` patches `_rpcFreePrologue` in `rpc.c` to return
  early when `IS_GSP_CLIENT(pGpu)`, so the FREE RPC (function 10) is
  never sent to the GSP.
- Combined with `heapSizeMBOverride = 512 MB` (enlarging the GSP heap to
  absorb leaked objects), this enabled full `cudaMalloc` + `cudaFree`
  at 76 GiB with **zero Xids** and a working `nvidia-smi` afterward
  (commit `90b31e4`).
- **What this confirmed:** the FREE-path crash is avoidable from the
  driver side. But the crash **also** fires via the
  `BUS_FLUSH_WITH_SYSMEMBAR` RPC (function 76, cmd `0x20800a70`) during
  teardown — that is a separate path that NOP-ing FREE does not stop.
  Function 76 is `GSP_RM_CONTROL` (the generic RM control channel), so it
  cannot be skipped without making the GPU useless. The FREE→NOP patch
  solves the FREE path but not the BUS_FLUSH path.
- **Limitation:** each skipped FREE leaks one GSP tracking object
  (~2500 slots in the 112 MB heap). At 76 GiB (19 chunks), the heap
  fills and the next ALLOC fails. The 512 MB override delays this but
  does not solve it for workloads with many alloc/free cycles.
- This was **source-only — never installed** on the running module. The
  loaded module at the time of the last session still sent FREE RPCs
  normally.
- A userspace alternative, `cudafree_fix.cpp`, is an `LD_PRELOAD` shim
  that intercepts `cudaFree` and redirects it to `cuMemFree` (driver
  API). It was reported to work at 52 GiB with zero Xids and to "repair"
  GSP state. A later source-check found that `cuMemFree` and `cudaFree`
  dispatch the same RPC (function 10), which weakens the "different code
  path" explanation — the observation may have been under a different
  build state. Needs re-testing under a known-correct build.

---

## 8. Open leads — ranked

### Lead 0 — the never-run data-path test
> Does `cudaMemcpy` into allocations above 40 GiB survive under the
> WPR2-fix build?

Never run under a working build. Pre-WPR2-fix it crashed at ~38 GiB (the
Gist failure). If it still crashes on the WPR2-fix build, the geometry
path does not solve the real problem and all teardown-workaround work is
moot. If it survives, the project has a real 80 GB data path and only
teardown remains.

### Lead C — REVISED: HAL-dispatch layout mismatch (the real bug)
The crash is a **field-layout mismatch** between object layouts selected by
SKU/class. The dispatch table is filled based on class/SKU info, but the
A/B dump on the 10 GB card showed the same host-sent class info in 40 GB
and 80 GB boots (8 GB card never A/B-tested — inference, not proof).
The bad class is derived by the GSP-RM from `fb_length` itself.

**Surgical fix target (driver-side, not firmware):** either NOP/redirect
the RPC that triggers the faulting method (the `BUS_FLUSH`/FREE path), or
patch the GSP-RM's `fb_length`→class table inside WPR2 (Lead D).

Next RE step: decode the range-check index at the three dispatch-table
fill sites (`0x4c007cc`, `0x4c00cec`, `0x4c087a0`) to identify which
class/SKU value selects the method-pointer layout vs the data-pointer
layout. READ-ONLY, no GPU.

### Lead D — Booter authenticated payload → FB memory write
cmpunlocker already injects authenticated *register* writes via
`_kgspSec2PostblTimingFillPayload` (executed by the Booter after RSA
verification, with MPU privilege). Open question: does the command format
support **FB memory writes**? If yes, patch the SKU table in WPR2 during
boot, before the GSP-RM reads it — bypassing the MPU entirely because the
Booter has internal access.

**Status:** the public gadget vocabulary has no FB-write gadget — only
`reg_write_indirect`. Several tail words (`0x00000cbd`, `0x00008e18`,
`0x0000ffbc`, `0x0000582d`, `0x0000815a`) are **unannotated**. The full
annotuted Booter disassembly (607 KB, 11,875 lines) and gadget atlas
(33 KB) exist in a private Discord archive — not in this repo. This is
the highest-effort path and the only permanent firmware-level fix.

### Lead E — A100 class fuse selector
The card already has `0x02` (A100 device_class) at `0x008202a0` and
`0x008202cc`, alongside `0x09` at `0x00820274`. A FUSE_OPT *enable* bit
that switches the primary class from `0x09`→`0x02` was never searched.
The `0x02` is already present; only selection is missing. Fuse-bank scan.

### Lead G — CE/DMA-initiated write to WPR2 (speculative)
MPU blocks *host*-initiated WPR2 access. Nobody has tested *device*-
initiated DMA (Copy Engine) into WPR2. The GSP self-writes WPR2 for its
own heap; a CE channel pointed at WPR2 might bypass the MPU. Pure
hypothesis, untried write primitive.

---

## 9. Recommended order of operations

```
1. Lead C-RE — decode the range-check index at the three dispatch-table
             fill sites (0x4c007cc / 0x4c00cec / 0x4c087a0) to identify
             which class/SKU value selects the method-pointer layout vs the
             data-pointer layout. READ-ONLY, no GPU. This is the controllable
             input and the key to the driver-side fix.

2. Lead 0  — run cudaMemcpy into allocations >40 GiB under the WPR2-fix
             build. Gate: does H2D-into-high survive?
             If NO  → the geometry path does not solve the real problem;
                      pivot fully to Lead C driver-side fix.
             If YES → data path works; only teardown remains.

3. Lead D  — Booter command format RE + radix3 WPR2 layout.
             The permanent firmware-level fix. Highest effort, last resort.
```

---

## 10. Deliverables

- **40 GB stable** on the 10 GB CMP 170HX (`0x2082`). Ships, clean,
  production-ready (`gpu_burn` clean, fold tests clean).
- A complete, evidence-based explanation of **why 80 GB is blocked**:
  - The SKU fuse (`0x2082` → `device_class=9, size_index=10` → 40 GB)
    sizes a GSP-RM internal allocation-tracking structure.
  - Total allocation > ~39.25–39.50 GiB corrupts that structure
    (allocation alone suffices; B1-RESERVE confirmed).
  - The fuse is OTP and appears to be read by the GSP-RM (three host
    overrides were all ignored), but we never traced the GSP-RM code
    that reads it. No host override affects it (A/B on 10 GB card:
    identical SKU/class in 40 GB and 80 GB; 8 GB card not tested).
  - The structure lives in WPR2; no available write primitive reaches it
    (MPU-blocked from host, Booter register-only, fwimage hash-blocked).
- The crash fully traced: HAL dispatch layout mismatch at `0x5b2b724`,
  wrong-method-dispatch reading `+0x1a8` (method pointer) as data.
- 79 GiB SM kernel fill + verify = ALL CLEAN (distinct physical memory
  confirmed, no aliasing, no fold).
- Clean OOM on overshoot (WPR2-overlap fix validated).
- The crash is deterministic (byte-identical crash dumps across runs).

---

## 11. The annotated Booter disassembly (a theory, not a probable solution)

The full annotated Booter disassembly
(`booter_load_ga100_dbg_seccode.annotated.fuc5_v2.asm`, 607 KB, 11,875
lines) and the gadget atlas (`register_gadget_atlas.md`, 33 KB) exist in
a private Discord archive of the `Consensus-Protocol/cmp170hx` project —
not in the public repo, not on the host.

**This is a lead worth pursuing if the material becomes available, not a
probable solution.** The public gadget vocabulary lists no FB-write gadget
— only `reg_write_indirect` (register write via CSB mailbox). Several
tail words are **unannotated**, and one *could* be an FB/PRAMIN/CE-DMA
write path. But they could equally be data, padding, or unrelated code.

If one of them is an FB-write gadget, the Booter could patch the SKU
table inside WPR2 during its privileged window — bypassing both the MPU
(Booter has LEVEL2) and the image-hash check (the GSP image is
unmodified; only the running WPR2 copy is patched after load). That
would be a permanent firmware-level fix. But without the annotated
listing, this is speculation.

**This should not be treated as the single remaining path or as evidence
that the verdict can be overturned.** See [`04_h2d-gsp-access-path_to-confirm.md`](04_h2d-gsp-access-path_to-confirm.md)
§4.

---

## 12. Recovery and safety contract

- One non-production card. eGPU, not a production host.
- Recovery: `sudo nvidia-smi -r` for Booter `0x29`; **cold power cycle**
  (unplug eGPU dock, wait for PSU light to die, replug) for `0xffff` and
  for stuck reset-required.
- Stop at first unexpected Xid, BAR loss, thermal excursion, or geometry
  mismatch. Preserve the kernel log from before CUDA init through recovery.
- Do NOT analyze `gsp_ga10x.bin` — the card loads `gsp_tu10x.bin`.

---

## 13. Host and build environment

- **Host:** `ggualerz@192.168.1.102` (key-based, no password)
- **cmpunlocker:** `~/cmpunlocker/driver`
- **Build:** `sudo CMPUNLOCKER_DRIVER_VERSION=610.43.02 CMPUNLOCKER_CARD_PROFILE=10gb CMPUNLOCKER_SKIP_RELOAD=1 bash build.sh`
- **Build stamp:** `sudo rm -f .build/open-gpu-kernel-modules-610.43.02/.cmpunlocker-stamp` (force re-patch)
- **CUDA:** `/usr/local/cuda/bin/nvcc`
- **Firmware:** `/lib/firmware/nvidia/610.43.02/gsp_tu10x.bin` (29 MB)
- **Test programs on host:** `~/gpu-tools/04-h2d-gsp/bin/`
- **Capstone:** `python3 -c "import capstone"` (RISC-V disassembler available)

---

## 14. Run results

All run data is in `data/raw/` (gitignored — preserved on disk only).

| Run | State | Result | Classification |
| --- | --- | --- | --- |
| `04-genesis-40g-kernel-1g-*` | 40 GiB, `hbm45`, 1 GiB | ALL CLEAN, no Xid | CONFIRMED control |
| `04-genesis-40g-h2d-1g-*` | 40 GiB, `hbmh2d`, 1 GiB | ALL CLEAN, no Xid | CONFIRMED control |
| `04-genesis-40g-stage-1g-*` | 40 GiB, `hbmstage`, 1 GiB | ALL CLEAN, no Xid | CONFIRMED control |
| `04-genesis-40g-kernel-37.5g-*` | 40 GiB, `hbm45`, 37.5 GiB | ALL CLEAN, no Xid | CONFIRMED control |
| `04-genesis-40g-h2d-37.5g-*` | 40 GiB, `hbmh2d`, 37.5 GiB | ALL CLEAN, no Xid | CONFIRMED control |
| `04-genesis-40g-h2d16-37.5g-*` | 40 GiB, `hbmh2d`, 16 MiB pieces | ALL CLEAN, no Xid | CONFIRMED control |
| `04-genesis-80g-kernel-1g-*` | 80 GiB, `hbm45`, 1 GiB | ALL CLEAN | CONFIRMED smoke test |
| `04-genesis-80g-kernel-76g-*` | 80 GiB, `hbm45`, 76 GiB, NVML polling | Incomplete (polling confound) | Invalid |
| `04-genesis-80g-kernel-nopoll-40g-*` | 80 GiB, `hbm45`, 40 GiB | 0-36 GiB clean; Xid 119→154 | CONFIRMED local fault |
| `04-genesis-gpu2-x4-80g-kernel-nopoll-36g-*` | 80 GiB, 2nd GPU, 36 GiB | ALL CLEAN, no Xid | CONFIRMED |
| `04-genesis-gpu2-x4-80g-kernel-nopoll-40g-*` | 80 GiB, 2nd GPU, 40 GiB | ALL CLEAN through 40; Xid 119×3→154 | CONFIRMED data + CONFLICT teardown |
| `04-genesis-gpu2-x4-80g-batched-kernel-nopoll-76g-*` | 80 GiB, batched, 76 GiB | ALL CLEAN 76 GiB; Xid 119×3→154 | CONFIRMED 76 GiB capacity |
| `04-genesis-gpu3-x4-80g-h2d-40g-*` | 80 GiB, 3rd GPU, `hbmh2d`, 40 GiB | Xid 119→1→119×2→154; ALL CLEAN to 40 GiB; data path survived GSP crash | CONFIRMED H2D fault reproduction |
| `04-genesis-gpu3-x4-80g-h2d-pinned-40g-*` | 80 GiB, pinned H2D, 40 GiB | Same crash, IDENTICAL registers, ~4× fewer RPCs | CONFIRMED: heap-churn hypothesis rejected |
| `04-genesis-gpu3-x4-80g-kernelmax-fixwpr2-*` | 80 GiB, fixed driver, `hbm_kernel_max` | 79 GiB ALL CLEAN; clean OOM; teardown Xid 119×3→154 | CONFIRMED 79 GiB + WPR2 fix |
| `04-genesis-gpu3-x4-80g-b1low-*` | 80 GiB, 8 GiB staging, 80 GiB cum H2D | ALL CLEAN, zero Xids, clean teardown | CONFIRMED: CE volume alone not sufficient |
| `04-genesis-gpu3-x4-80g-b1reserve-*` | 80 GiB, 40 GiB alloc, 64 GiB H2D chunk 0 | Data ALL CLEAN; teardown faulted | CONFIRMED: allocation alone corrupts state |
| `04-genesis-gpu3-x4-80g-b1cross-*` | 80 GiB, alloc threshold bisection | Threshold 157→158 chunks (39.25→39.50 GiB) | CONFIRMED: ~40 GiB structure limit |

---

## 15. Upstream provenance

Primary source: [CMP 170HX 80GB geometry: DMA above ~40 GiB crashes GSP](https://gist.github.com/cuddylac997/c3d80faa2430e3650cd934eda5fd65a9)
(preserved Gist commit `2f65500f3c81609f7d46522b1c7df5dea16dd5c6`)

Preserved source hashes (`src/upstream-gist/`):
```
17703ae975488967c6dccc5129dd8784c8083193282ccc6f336137fc25683698  hbm45.cu
de067d1cd078535fbe1e78457b7dd70174b9980d39fa400c0ff0fef27025431a  hbmh2d.cu
b47a7cde8347d133e742e2e450fb26b9d4bc6f7419f674b199929388082784b  hbmstage.cu
41b0a90947817d57de81e1ce45cc5bf485f706bec6cba22e7f7a6e843bc1cddb  170hx-gsp-dma-report.md
```

Built with CUDA 13.1: `nvcc -O2 -arch=sm_80 -Xcompiler=-U_GNU_SOURCE`

Test environment:
```
Host:             genesis
GPU:              NVIDIA CMP 170HX, 10de:2082, GA100 rev a1
Attachment:       external PCIe eGPU through USB4
Successful run:   link width x4, negotiated speed 2.5 GT/s
Driver:           610.43.02, patched open kernel module
CUDA toolkit:     13.1
Kernel:           7.0.0-30-generic
cmpunlocker base: 8f3c82f438e474857edfa464d01fc8d5eaf96c5c
GSP firmware:     gsp_tu10x.bin (29 MB) — GA100 loads THIS, not gsp_ga10x.bin
```

---

## 16. Directory structure

```
research-report.md             this document
04_h2d-gsp-access-path_to-confirm.md                  open questions and unverified inferences

data/environment/              host identity, coherent patch, boot geometry, module backup
data/raw/                      27 run directories (kernel logs, crash dumps — gitignored, on disk only)

src/                           test programs
  hbm_kernel_max.cu            allocate-to-OOM + kernel fill + verify + overshoot probe
  hbm_kernel_batched.cu        batched kernel fill/verify, single D2H verdict, no cudaFree
  hbmh2d_b1reserve.cu          79 GiB alloc + cumulative H2D into chunk 0
  hbmh2d_b1low.cu              bounded staging alloc + cumulative H2D
  hbmh2d_memtype.cu            pinned vs pageable H2D test
  test_vmm_free.cu             VMM cuMemCreate/cuMemRelease free-path test
  test_vmm_free2.cu            extended VMM free-path test
  test_free_threshold.cu       alloc N GiB + fill + verify + cudaFree (threshold test)
  test_cumemfree.cu            cuMemFree workaround test
  test_cumemfree_chain.cu      cuMemFree then cudaFree chained test
  test_cumemfree_ceiling.cu    cuMemFree ceiling test
  test_explicit_free.cu        explicit free test
  test_natural_exit.cu         natural process exit test (no cudaFree)
  cudafree_fix.cu              LD_PRELOAD shim: cudaFree → cuMemFree redirect
  upstream-gist/               exact upstream Gist sources used for comparison
    hbm45.cu                   upstream kernel-fill test
    hbmh2d.cu                  upstream H2D test
    hbmstage.cu                upstream staging test
    170hx-gsp-dma-report.md    upstream Gist report

patches/                       driver injection scripts
  inject_sec2_shadow.py        SEC2 shadow register override patch
  inject_devid.py              device ID override patch (disabled, no effect)
  inject_staticinfo_dump.py    A/B static-info dump patch
  parse_staticinfo_dump.py     parser for A/B dump output
  inject_free_skip.py          FREE→NOP kernel patch (rpc.c _rpcFreePrologue)
  cudafree_fix.cpp             LD_PRELOAD shim source (cudaFree → cuMemFree)
  sku_func.c                   PRAMIN scan + SKU table patch (C, driver-side)
  sku_fwimage_patch.c          fwimage pre-load patch (C, driver-side)
  insert_fwimage_patch.py      Python inserter for fwimage patch
  fix_build_fwimage.py         build.sh modifier for fwimage patch
  build_snippet.sh             build snippet showing patch invocation

tools/                         firmware analysis scripts (key RE tools only)
  analysis/
    find_rpc_table.py          RPC dispatch table finder (1855 entries at 0x407b638)
    trace_badptr2.py           bad-pointer tracer (crash method 0x5b2b724)
    shadow_bigpicture.py       SEC2 shadow register big-picture analysis
```

| Document | Content |
|---|---|
| `research-report.md` | This document — the consolidated narrative |
| `04_h2d-gsp-access-path_to-confirm.md` | Open questions and unverified inferences |
| `ab_dump/ANALYSIS.md` | A/B static-info dump diff (40 GB vs 80 GB) |
| `data/environment/` | Host identity, coherent patch, boot geometry, module backup |
| `data/raw/` | 27 run directories with kernel logs and crash dumps (gitignored — on disk only) |
| `src/` | Test programs (hbm_kernel_max, hbmh2d variants, test_vmm_free, cudafree_fix, etc.) |
| `src/upstream-gist/` | Exact upstream Gist sources used for comparison |
| `patches/` | Driver injection scripts (SEC2 shadow, staticinfo dump, fwimage patch, free-skip, cudafree_fix) |
| `tools/analysis/` | Key firmware RE scripts (find_rpc_table, trace_badptr, shadow_bigpicture) |
