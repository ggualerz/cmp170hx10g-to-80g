# TO-CONFIRM — Open questions and unverified inferences

This file tracks claims that were stated with more confidence than the
evidence supports. Each item lists what is actually known, what is
inference, and what would be needed to confirm or refute it.

---

## 1. Is the 40 GB limit really derived from the OTP fuse FUSE_PCIE_DEVIDA (0x2082)?

**What is proven:**
- `FUSE_PCIE_DEVIDA` (register `0x008204D8`) reads `0x00002082` on this card.
- The GSP-RM ignores the host-provided PCI device ID from three independent
  override paths: RPC `SET_GUEST_SYSTEM_INFO` (`PCIDeviceID` field),
  `NV_XVE_ID` register write, and PCI config mirror disable. All three left
  the crash unchanged.
- A SKU table exists in `gsp_tu10x.bin` at `.fwimage` offset `0x23460`:
  `device_class=9, size_index=10` (40 GB) for `0x2082`.
- `FEAT_OVR` registers are 1-bit enable/disable masks, not multi-bit value
  overrides — they cannot change `0x2082` to `0x20B5`.
- The `FUSE_PCIE_DEVIDA` register is **undocumented** (not in the published
  `dev_fuse.h` header), so its exact role in the GSP-RM's init path is
  inferred, not traced.

**What is inference:**
- That the GSP-RM reads `0x008204D8` **directly** as the source of the
  device ID. The three failed overrides prove the GSP-RM does not use the
  host-provided or XVE-mirrored device ID. But "it reads the OTP fuse
  directly" is the remaining hypothesis, not a traced code path. The GSP-RM
  could derive the device class from a different fuse, a combination of
  fuses, or an internal computation we have not identified.
- The A/B static-info dump (section 5 of the research report) showed that
  every SKU/identity/class field in `GspStaticConfigInfo` and
  `GspSystemInfo` is **identical** between 40 GB and 80 GB boots **on the
  10 GB card**. This means the GSP-RM computes the same identity from the
  same inputs in both cases on this card — but it does not prove which
  specific input (fuse, PCI hardware, or internal computation) is the
  load-bearing one. Additionally, the 8 GB card (0x20C2 → 64 GB stable)
  was **never A/B-tested**. The 8 GB card runs 64 GB stably with the same
  GSP-RM and driver, so its host-side static config may differ in a way
  that the 10 GB card's does not. Without an A/B on the 8 GB card, we
  cannot fully exclude that a host-visible field difference exists for
  some SKU paths.

**What would confirm it:**
- Trace the GSP-RM init code that reads `0x008204D8` and follows the value
  through to the SKU table lookup. This requires either a debug GSP
  firmware (with `.fwlogging` strings) or RE of the GSP-RM boot code's
  fuse-read path. The `gsp_tu10x.bin` section table is zeroed and the
  `.logging_const` strings are stripped from the production firmware.
- Alternatively: if a fuse bit or register exists that can switch the
  primary device class from `0x09` to `0x02` (the A100 class already
  present at `0x008202a0` / `0x008202cc`), and flipping it changes the
  40 GB behavior, that would confirm the fuse→class→limit chain.

---

## 2. Are the SEC2 shadow registers (0x840a48 / 0x840a94) really PRNG state?

**What is observed:**
- `0x00840a48` and `0x00840a94` change values between reads spaced seconds
  apart. `0x840a48` was observed incrementing `0x09 → 0x0a → ... → 0x10`
  over ~16 seconds. `0x840a94` jumps erratically.
- `0x840a94` read back `0xffffffff` on reproduction attempts (the write
  appears rejected or state-locked).

**What is NOT confirmed (misattribution found in earlier notes):**
- `SEC2_SHADOW_OVERRIDE_RESULTS.md` (2026-08-29) claimed a boot-time write
  to these registers eliminated the 40 GB wall once, and attributed the
  79 GiB clean result to the SEC2 patch. No preserved raw run supports
  this: the matching preserved run (`kernelmax-fixwpr2-20260828T124248Z`)
  predates the SEC2 patch, has no SEC2 write lines in its kernel log, and
  used the WPR2-overlap fix. The one SEC2 reproduction attempt (v5 build)
  crashed during the data phase at ~184 s. The 79 GiB result is a
  WPR2-fix result, not a SEC2 result.

**What is inference:**
- The "PRNG" label. The observation is only that these registers change at
  runtime. Calling them "PRNG state" is one explanation. The registers could
  also be:
  - counters or timestamps consumed by the GSP-RM,
  - state machines whose phase at boot determines init behavior,
  - registers with a race condition between the host write and the GSP's
    own init sequence,
  - or something else entirely.
- The `SEC2_SHADOW_BIGPICTURE.md` document itself flagged the PRNG
  conclusion as **UNPROVEN**. A later session reclassified it as confirmed
  PRNG, but that reclassification was based on the same observations
  (values change + effect non-deterministic), not on tracing a PRNG
  algorithm in the firmware.

**What would confirm or refute it:**
- Trace the GSP-RM code that reads/writes `0x840a48` and `0x840a94`. If
  the code implements a PRNG update (LFSR, linear congruential, etc.),
  the label is confirmed. If it reads them as configuration, the label is
  wrong.
- A deterministic boot-time write to these registers that reliably
  eliminates the 40 GB wall would refute the "PRNG non-determinism"
  conclusion. No such result exists in the preserved data.

---

## 3. The assessment: "80 GB is not achievable with any tested mechanism"

**This is not a proven statement.** It is the current best assessment
given the approaches that have been tried and failed. It should be read
as "no mechanism we have tested works," not as a proof that 80 GB is
impossible.

**What undermines the assessment:**
- The 79 GiB allocation + SM fill + verify ALL CLEAN run is real
  (31/31 chunks, zero failures, clean OOM) — but it was achieved by the
  WPR2/PMA-overlap fix on coherent geometry, **without** the SEC2 patch.
  It does not lift the 40 GB GSP-RM allocation limit; the limit is a
  separate firmware-internal mechanism. The earlier attribution of this
  result to the SEC2 shadow write was a misattribution (see §2).
- The FREE→NOP driver patch enabled `cudaFree` at 76 GiB with zero Xids,
  proving the FREE-path crash is avoidable from the driver side. However,
  the crash also fires via the `BUS_FLUSH_WITH_SYSMEMBAR` RPC (function 76),
  which is a separate path — so NOP-ing FREE alone does not solve the full
  teardown. See §7.1 of the research report.
- The `cuMemFree` / `LD_PRELOAD` workaround was reported to survive
  freeing > 40 GiB, but the source-check showed `cuMemFree` and
  `cudaFree` dispatch the same RPC (function 10), weakening the
  "different code path" explanation. Needs re-testing under a known-correct
  build.
- The annotated Booter disassembly (not in this repo) could contain an
  FB/PRAMIN write gadget among the unannotated tail words. This is
  speculative but untested.

---

## 4. The annotated Booter disassembly

This should **not** be presented as the single remaining path or as
evidence that the assessment can be overturned. It is a **theory** about
where a potential write primitive might exist, based on unannotated
tail words in the public gadget vocabulary. The full annotated
disassembly and gadget atlas exist in a private Discord archive of the
`Consensus-Protocol/cmp170hx` project and are not in this repo.

The public gadget vocabulary's only write primitive is
`reg_write_indirect` (register write, not FB write). The unannotated
tail words could equally be data, padding, or unrelated code. This is a
lead worth pursuing if the material becomes available, not a probable
solution.